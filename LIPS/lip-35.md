---
lip: 35
title: Staking Router v3
status: Proposed
author: Maksim Kuraian (@mkurayan) , KRogLA (@KRogLA), Alexander Kolesnikov (@eddort), Anna Mukharram (@Amuhar)
discussions-to: To be updated before merge
created: 2026-05-22
updated: 2026-05-22
---

# LIP-35. Staking Router v3

## Table of Contents

- [LIP-35. Staking Router v3](#lip-35-staking-router-v3)
  - [Table of Contents](#table-of-contents)
  - [Simple Summary](#simple-summary)
  - [Motivation](#motivation)
- [Specification](#specification)
  - [Glossary](#glossary)
  - [Accounting](#accounting)
    - [The 32 ETH Problem](#the-32-eth-problem)
    - [New Accounting Model](#new-accounting-model)
      - [Accounting oracle report changes](#accounting-oracle-report-changes)
      - [CL Validators Balance Calculation Algorithm](#cl-validators-balance-calculation-algorithm)
      - [Pending Deposits Calculation Algorithm](#pending-deposits-calculation-algorithm)
    - [From Transient to Pending](#from-transient-to-pending)
    - [Deposit Tracking](#deposit-tracking)
    - [Rewards Calculation](#rewards-calculation)
      - [Rewards Distribution](#rewards-distribution)
    - [Migration](#migration)
      - [Lido Migration](#lido-migration)
      - [StakingRouter Migration](#stakingrouter-migration)
      - [Data Integrity](#data-integrity)
    - [Sanity Checks](#sanity-checks)
      - [AO CL Balance Decrease](#ao-cl-balance-decrease)
          - [First 36 Days Problem](#first-36-days-problem)
      - [AO CL Balances Consistency](#ao-cl-balances-consistency)
      - [AO Balance Increase Per Day](#ao-balance-increase-per-day)
      - [AO Exited Validators by Staking Module](#ao-exited-validators-by-staking-module)
      - [VEBO Maximum Eth Amount To Exit](#vebo-maximum-eth-amount-to-exit)
    - [Legacy API](#legacy-api)
  - [Consolidation](#consolidation)
    - [Consolidation EasyTrack](#consolidation-easytrack)
    - [Consolidation Migrator](#consolidation-migrator)
      - [Module Interaction](#module-interaction)
    - [Consolidation Message Bus](#consolidation-message-bus)
      - [Execution delay](#execution-delay)
      - [Executor Bot](#executor-bot)
    - [Consolidation Gateway](#consolidation-gateway)
    - [Withdrawal Vault](#withdrawal-vault)
  - [Deposits](#deposits)
    - [Depositable ETH Pull Model](#depositable-eth-pull-model)
      - [Lido](#lido)
      - [Staking Router](#staking-router)
    - [Predeposit flow](#predeposit-flow)
    - [Top-up flow](#top-up-flow)
    - [Depositor Bot](#depositor-bot)
      - [Deposit Module Selection](#deposit-module-selection)
      - [Stage 1. Predeposits to modules with `0x02` keys](#stage-1-predeposits-to-modules-with-0x02-keys)
      - [Stage 2. Top-ups to modules with `0x02` keys or deposits to `0x01` modules](#stage-2-top-ups-to-modules-with-0x02-keys-or-deposits-to-0x01-modules)
      - [Depositor Bot top-up flow for `0x02` key modules](#depositor-bot-top-up-flow-for-0x02-key-modules)
      - [Top-up key selection for CMv2](#top-up-key-selection-for-cmv2)
      - [Top-up key selection for the future `0x02` version of CSM](#top-up-key-selection-for-the-future-0x02-version-of-csm)
    - [TopUpGateway](#topupgateway)
      - [Proof building rules](#proof-building-rules)
      - [Validator State Validation](#validator-state-validation)
      - [Validator Pending Balance](#validator-pending-balance)
      - [Validator Top-Up Limit Calculation](#validator-top-up-limit-calculation)
      - [Top-Up Limits](#top-up-limits)
      - [Top-up Precondition and Pause](#top-up-precondition-and-pause)
    - [Staking Router](#staking-router-1)
      - [Staking Router Configuration](#staking-router-configuration)
      - [Request limits](#request-limits)
      - [Module stake allocation](#module-stake-allocation)
    - [Staking Modules](#staking-modules)
      - [CMv2](#cmv2)
      - [`0x02` version of CSM](#0x02-version-of-csm)
  - [Validators Exits](#validators-exits)
    - [Validators Exit Bus Oracle (VEBO)](#validators-exit-bus-oracle-vebo)
      - [Report format](#report-format)
      - [Sanity checker](#sanity-checker)
    - [Off-chain Validators Exit Oracle](#off-chain-validators-exit-oracle)
      - [Deposit reserve](#deposit-reserve)
      - [Consolidation](#consolidation-1)
      - [Operator weight in the CMv2 module](#operator-weight-in-the-cmv2-module)
  - [Stake Rebalancing](#stake-rebalancing)
    - [Deposit Reserve](#deposit-reserve-1)
      - [Proposed solution](#proposed-solution)
      - [Buffered Ether Allocation](#buffered-ether-allocation)
    - [Edge Case — Last Withdrawals](#edge-case--last-withdrawals)
  - [Module Shares Easy Track](#module-shares-easy-track)
    - [Limits](#limits)
    - [Algorithm](#algorithm)
    - [Motion creation](#motion-creation)
      - [Motion enacting](#motion-enacting)
    - [Required changes to StakingRouter](#required-changes-to-stakingrouter)
  - [Proposed params and roles](#proposed-params-and-roles)
    - [Lido](#lido-1)
    - [LidoLocator](#lidolocator)
    - [StakingRouter](#stakingrouter)
    - [AccountingOracle](#accountingoracle)
    - [ValidatorsExitBusOracle](#validatorsexitbusoracle)
    - [WithdrawalVault](#withdrawalvault)
    - [DepositSecurityModule](#depositsecuritymodule)
    - [OracleReportSanityChecker](#oraclereportsanitychecker)
    - [ConsolidationGateway](#consolidationgateway)
    - [ConsolidationGateway CircuitBreaker registration](#consolidationgateway-circuitbreaker-registration)
    - [ConsolidationBus](#consolidationbus)
    - [ConsolidationMigrator](#consolidationmigrator)
    - [TopUpGateway](#topupgateway-1)
    - [TopUpGateway CircuitBreaker registration](#topupgateway-circuitbreaker-registration)
    - [TriggerableWithdrawalGateway](#triggerablewithdrawalgateway)
    - [EasyTrack](#easytrack)
  - [Security Considerations](#security-considerations)
    - [On-chain proofs as the trust anchor for top-ups and consolidations](#on-chain-proofs-as-the-trust-anchor-for-top-ups-and-consolidations)
    - [Authorized Depositor Bot for Top-Ups](#authorized-depositor-bot-for-top-ups)
    - [Deposit reserve and withdrawal demand](#deposit-reserve-and-withdrawal-demand)
  - [References](#references)

## Simple Summary

Staking Router v3 upgrades Lido's core protocol for [EIP-7251](https://eips.ethereum.org/EIPS/eip-7251) (MaxEB). Validator accounting moves from per-validator counts to direct balance tracking, rewards are distributed per-module balance, and the deposit flow switches from a push model orchestrated by the Lido contract to a pull model where the Staking Router withdraws ETH from Lido as needed. A new `TopUpGateway` enables top-ups of `0x02` validators up to 2048 ETH, secured by on-chain Merkle-proof verification of validator state.

A consolidation pipeline enables stake migration from Curated Module v1 to v2, and the validator exit flow becomes key-type-aware and balance-based, with sanity checks bound by total effective balance rather than validator count.

A deposit reserve protects a portion of buffered ether for CL deposits, preventing withdrawal demand from consuming ETH needed for stake rebalancing and initial deposits during the CMv1 to CMv2 migration.

A dedicated Easy Track factory streamlines module share management within pre-defined bounds.

## Motivation

Staking Router v3 is the foundational infrastructure upgrade that serves as the base layer upon which [LIP-33 (CSM v3 and CM v2)](lip-33.md) is built. It unlocks a more flexible and efficient protocol, both technically and operationally. With these changes, the protocol will be able to reallocate stake between modules via consolidation operations, support new deposits into large validators, naturally streamlining the network, and lay the groundwork for smarter, leaner validator management overall.

**A new, `0x02`-ready, accounting model.** Although already supported by the stVaults architecture, this is a fundamental shift for Lido Core. Currently, the Lido protocol handles critical aspects of validator accounting — deposits, rewards, withdrawals — through a unit-based approach where 1 validator equals 32 ETH. A balance-based accounting model is essential for supporting large validators and consolidations because it allows the protocol to treat validators as flexible balances rather than fixed units, enabling seamless consolidation and validator top-ups. For this to happen, a significant rework of several key components of the on-chain protocol is required.

**Stake migration from CMv1 to CMv2.** The consolidation process would provide a secure and transparent way to quickly migrate stake from the old Curated Module v1 to the new Curated Module v2 with minimal stake inefficiency.

**Deposit reserve for reliable stake rebalancing.** Historically, stake rebalancing between modules through new deposits and withdrawals has been slow and unreliable — SDVT took over 1.5 years to reach its target share. A deposit reserve mechanism guarantees ETH availability for migration deposits and new module onboarding, regardless of withdrawal demand.

# Specification

## Glossary

- **EL** — Execution Layer. The Ethereum layer that processes transactions, executes smart contracts, and maintains Ethereum’s account and contract state.
- **CL** — Consensus Layer. The Ethereum layer that runs proof-of-stake, coordinates validators, chooses the canonical chain, and provides finality.
- **MaxEB** — Maximum Effective Balance. Raised by [EIP-7251](https://eips.ethereum.org/EIPS/eip-7251) from 32 ETH up to 2048 ETH for validators with `0x02` withdrawal credentials.
- **WC** — Withdrawal Credentials. The 32-byte field on a validator that determines where its stake exits to and which key type it uses. All Lido core validators use credentials pointing to the `WithdrawalVault` contract.
- **`0x01` / `0x02` keys** — validator key types, distinguished by withdrawal credentials prefix. `0x01` keys are capped at 32 ETH effective balance; `0x02` keys support effective balance up to 2048 ETH (MaxEB) and allow top-ups and consolidations.
- **Predeposit** — the minimal deposit (32 ETH) required to activate a validator. Required for both `0x01` and `0x02` keys.
- **Top-up** — an additional deposit to an already-activated `0x02` validator, raising its balance toward the 2048 ETH cap.
- **Consolidation** — an [EIP-7251](https://eips.ethereum.org/EIPS/eip-7251) operation that merges a source validator's balance into a target `0x02` validator.
- **CMv1 / CMv2** — Curated Module v1 (the existing curated module with 0x01 withdrawal credentials, also known as Node Operators registry, or simply Curated Module) and Curated Module v2 (the new curated module supporting 0x02 withdrawal credentials only).
- **CSM** — Community Staking Module, Lido's permissionless staking module.
- **SDVT** — Simple DVT module, the staking module that runs distributed validators. Uses the same code base as CMv1.
- **Deposit Reserve** — a portion of Lido's buffered ether protected from withdrawal demand and reserved for CL deposits, ensuring stake rebalancing and migration deposits are not blocked.
- **AO** — Accounting Oracle. The oracle that reports validator balances, rewards, and other accounting data to the protocol.
- **VEBO** — Validators Exit Bus Oracle. The oracle that publishes validator exit requests.
- **DSM** — Deposit Security Module. The contract and a distributed off-chain service that gates Lido initial 32 ETH deposit that activates a validator and can be paused by Council guardians.
- **TWG** — TriggerableWithdrawalGateway. The contract that submits [EIP-7002](https://eips.ethereum.org/EIPS/eip-7002) triggerable withdrawal requests on behalf of the protocol.

## Accounting

### The 32 ETH Problem

Before the [Pectra](https://eips.ethereum.org/EIPS/eip-7600) upgrade, every Ethereum validator had a fixed maximum effective balance of 32 ETH. This allowed Lido to use a simple accounting model: it was enough to count the number of validators and multiply by 32 to get the total stake.

[EIP-7251](https://eips.ethereum.org/EIPS/eip-7251) introduced the new type of validator withdrawal credentials - `0x02` with `MaxEB = 2048 ETH`. For these validators, an effective balance can range from 32 to 2048 ETH. A validator with 2048 ETH generates 64 times more rewards than a validator with 32 ETH, but the old accounting system in the Lido protocol treated them equally.

As a result, the current validator-count–based accounting is incompatible with large validators in several aspects:

- **Rewards calculation** — the protocol doesn’t know how much ETH is actually working on the Consensus Layer
- **Stake distribution** — allocation between modules is based on validator count, not actual balance
- **Tracking `totalPooledEther`** — the key metric for stETH rate calculation becomes inaccurate

The solution is to transition from counting validators to direct balance accounting.

### New Accounting Model

Currently, in the `Lido` contract the protocol stores two values: `clValidators` (number of validators on the CL) and `clBalance` (their total balance). The validator count is used to calculate transient balance and perform security checks.

It is proposed to replace `clBalance` and `clValidators` in Lido with `clValidatorsBalance` and `clPendingBalance`. The oracle report structure changes accordingly — instead of total balance, it now delivers balances split by state.

#### Accounting oracle report changes

```diff
- clBalanceGwei
+ clValidatorsBalanceGwei
+ clPendingBalanceGwei
+ stakingModuleIdsWithUpdatedBalance[]
+ validatorBalancesGweiByStakingModule[]
```

Total protocol balance on the CL:

```
totalClBalance = clValidatorsBalance + clPendingBalance
```

> Note: For the `clValidatorsBalanceGwei` and `clPendingBalanceGwei` calculations, the oracle uses [Keys API](https://github.com/lidofinance/lido-keys-api). When querying Keys API, the oracle verifies that the returned data corresponds to an EL block number greater than or equal to the reference block, ensuring that the response includes the most up-to-date state.

#### CL Validators Balance Calculation Algorithm

1. **Data retrieval:** get the active validator set (with balances) from the Consensus Layer, and the list of Lido keys from the Keys API.
2. **Consistency check:** the number of Lido keys returned by the Keys API must be at least equal to the count of deposited validators reported by the Lido contract (`depositedValidators`). If fewer keys are returned, revert.
3. **Match keys to validators:** for every Lido key, look it up in the CL validator set:
   - If the key is present on the CL, it belongs to an active Lido validator and is included in the report.
   - If the key is not yet on the CL, it is pending (deposited on the Execution Layer, not yet activated on the Consensus Layer) and is skipped here — its balance is handled separately by the pending-balance calculation.
4. **Sum balances:** `clValidatorsBalance` is the sum of balances across all matched active Lido validators.

#### Pending Deposits Calculation Algorithm

The pending balance covers every pending CL deposit attributable to a Lido key, whether the target is a key awaiting activation (a new validator) or an already-active validator (a top-up).

1. **Data retrieval:** get pending deposits and the active validator set from the Consensus Layer, Lido keys from the Keys API, and Lido withdrawal credentials from the Lido contracts.
2. **Consistency check:** the number of Lido keys returned by the Keys API must be at least equal to the count of deposited validators reported by the Lido contract (`depositedValidators`). If fewer keys are returned, revert.
3. **Find Lido keys awaiting activation:** compare Lido keys against the CL validator set — keys not yet present on the CL are considered _pending_ (deposited on the Execution Layer, not yet activated on the Consensus Layer).
4. **Attribute pending deposits to Lido keys:** walk over pending deposits in queue order and locate each deposit's target Lido key:
   - If the pubkey matches an **active Lido validator** (already on CL), credit the deposit as a top-up. No BLS-signature or front-run check is needed because the validator's withdrawal credentials are already immutable on the CL.
   - If the pubkey matches a **pending Lido key** (not yet on CL):
     - Skip deposits with an invalid BLS signature.
     - The first BLS-valid deposit for the key determines its status: if it points to Lido withdrawal credentials, the key is accepted and the deposit is kept; otherwise the key is considered front-run and all of its deposits are discarded.
     - Subsequent deposits to an already-accepted pending key are added to that key's group.
5. **Sum balances:** `clPendingBalance` is the sum of all deposit amounts attributed in step 4 (both pending-key activations and active-validator top-ups).

### From Transient to Pending

**Currently**, there is a “transient balance” — a virtual value compensating for the delay between sending a deposit and its appearance on the Consensus Layer:

```
transientBalance = (depositedValidators - clValidators) × 32 ETH
```

This construct was necessary because a deposit could “temporary disappear” between the Execution Layer and the Consensus Layer for several hours or even days. The oracle didn’t see these funds, but the protocol had to account for them in `totalPooledEther`.

**Proposed:** [EIP-6110](https://eips.ethereum.org/EIPS/eip-6110), introduced in Pectra Hard Fork,  eliminates this problem. Deposits from the Execution Layer enter the Consensus Layer pending queue in the same block:

> “Validator deposits list supplied in a block is obtained by parsing deposit contract log events emitted by each deposit transaction included in a given block.” — [EIP-6110](https://eips.ethereum.org/EIPS/eip-6110)

The oracle always sees the complete picture: active validators in `clValidatorsBalance`, pending deposits in `clPendingBalance`. Transient balance is no longer needed — this simplifies the system and eliminates a potential source of discrepancies between the actual CL state and protocol data on EL.

### Deposit Tracking

Each accounting report only covers deposits made up to its reference slot. Any deposits made after that slot are not yet in the report, so the protocol should track them on its own until the next report picks them up.

**Currently**, deposits are tracked through validator counters. When ETH moves from the buffer to the Deposit Contract:

```diff
- buffered -= 32 ETH
- depositedValidators += 1
```

The transient balance `(depositedValidators - clValidators) × 32 ETH` compensates for deposits in transit.

**Proposed:** deposits are tracked through balance using two counters. When ETH moves from the buffer to the Deposit Contract:

```diff
+ buffered -= amount
+ depositedSinceLastReport += amount
+ depositedForCurrentReport += amount
```

Where:

- `depositedSinceLastReport` — total deposits since the last oracle report reference slot across all frames (includes deposits made after the current reporting frame)
- `depositedForCurrentReport` — deposits that occurred between the last report reference slot and the current frame's reference slot (i.e., deposits the oracle should have observed; excludes deposits made after the current reporting frame)

```
                                                               NOW
                     ┌─ depositedSinceLastReport ────────────┐ ↓
                     │─ depositedForCurrentReport ─┐         │
      │○○○○○○○○○○○○○○│○●●○○R○○○●○○●○│○○●●●○○●○●○○○○│○○●●○○●○○●○○○○│
      ┆         lastReport-↑       currentRefSlot-↑└────⁠┬────┘
      ┆              ┆ currentReportFrame-↓        ┆    └depositedNextReport
      ⁠║   frame X    ⁠║   frame X+1  ⁠║   frame X+2  ⁠║   frame X+3  ⁠║

       R - report transaction slot
       ● - slot with deposits
       ○ - empty slot
       ⁠║ - frame refSlot
```

The new `getBalanceStats()` method provides full balance model data:

```solidity
interface ILido {
  /// @notice Returns current balance statistics
  /// @return clValidatorsBalanceAtLastReport Sum of validator's active balances in wei
  /// @return clPendingBalanceAtLastReport Sum of validator's pending deposits in wei
  /// @return depositedSinceLastReport Deposits made since last oracle report reference slot
  /// @return depositedForCurrentReport Deposits made between the last oracle report reference slot and the current frame's reference slot
  function getBalanceStats()
    external
    view
    returns (
      uint256 clValidatorsBalanceAtLastReport,
      uint256 clPendingBalanceAtLastReport,
      uint256 depositedSinceLastReport,
      uint256 depositedForCurrentReport
    );
}
```

With this transition, the following deposit-related members are no longer needed and are proposed to be removed from Lido:

- `unsafeChangeDepositedValidators(uint256 _newDepositedValidators)` and the associated `UNSAFE_CHANGE_DEPOSITED_VALIDATORS_ROLE` — no longer required due to the transition to balance-based deposit tracking

### Rewards Calculation

**Currently**, the principal balance (total balance of validators in the previous report and deposits made since then) is calculated taking into account new validators:

```diff
- principalClBalance = prev.clBalance + (report.clValidators - prev.clValidators) × 32 ETH
- clRewards = _report.clBalance + update.withdrawalsVaultTransfer - update.principalClBalance
```

The formula compensates for deposits “in transit” but is tied to the 32 ETH assumption.

**Proposed:** the principal is taken directly from storage:

```diff
+ principalClBalance = prev.clValidatorsBalance + prev.clPendingBalance + prev.depositedSinceLastReport;
+ clRewards = (report.clValidatorsBalance + report.clPendingBalance) + update.withdrawalsVaultTransfer - principalClBalance
```

#### Rewards Distribution

**Currently**, a module’s share of rewards is determined by its active validator count:

```diff
- moduleShare = activeValidatorsCount / totalActiveValidators
```

**Proposed:** the share is determined by the module’s active validator balance:

```diff
+ moduleShare = moduleActiveBalance / totalActiveBalance
```

Where `moduleActiveBalance` is the module’s `validatorsBalanceGwei`, stored in the StakingRouter and updated by the oracle. As part of the main Accounting Oracle report phase, the oracle submits per-module validator balance data, which is routed to the `StakingRouter` via the following function:

```solidity
function reportValidatorBalancesByStakingModule(
    uint256[] calldata _stakingModuleIds,
    uint256[] calldata _validatorBalancesGwei,
) external;
```

The subsequent fee calculation has not changed: module and treasury fees are calculated from the share multiplied by the corresponding basis points from the module configuration. If a module is in the Stopped status, its fee is redirected to the treasury.

A module with one validator at 2048 ETH now receives a fair share of rewards corresponding to its contribution to the total stake, not an undervalued share of “one validator among many.”

### Migration

The migration aims to preserve data integrity and ensure correct rewards calculation in the first report after the upgrade.

#### Lido Migration

```diff
  depositedValidators
+ clValidatorsBalance
+ clPendingBalance
- clBalance
- clValidators
```

**Current state:** a single packed storage slot containing `clBalance` and `clValidators`.

**Target state:** a single packed storage slot containing `clValidatorsBalance` and `clPendingBalance`.

**Migration procedure:**

1. Read legacy state:
   - `clBalance`
   - `clValidators`
   - `depositedValidators`
2. Compute the transient balance as
   `transientBalance = (depositedValidators − clValidators) × 32 ETH`.
3. Populate the new state:
   - `clValidatorsBalance = clBalance`
   - `clPendingBalance = transientBalance`

The transient balance becomes the pending balance — semantically, it is the same thing (ETH on the way to activation), but now the oracle will report the actual value instead of a calculated one.

#### StakingRouter Migration

**Before:** modules store only validator counters (`depositedValidatorsCount`, `exitedValidatorsCount`).

**After:** each module stores `validatorsBalanceGwei`.

For each legacy module during migration:

1. Get deposited and exited validator counts
2. Calculate active: `activeCount = depositedValidators - exitedValidators`
3. Calculate initial balance: `validatorBalanceGwei = activeCount × 32 ETH`

The sum of `validatorsBalanceGwei` across all modules is written to the shared `RouterStateAccounting`. The next oracle report will update these values to actual balances from the Consensus Layer.

#### Data Integrity

The first report after migration is correct by construction:

- `principalClBalance` includes the migrated pending balance
- New deposits after migration are visible to the oracle in `clPendingBalance`
- Old transient stake is not counted as rewards

### Sanity Checks

The list of checks has been updated following the **Pectra hard-fork** and the subsequent expansion of parameters passed to the Validator Exit Bus Oracle (VEBO) and Accounting Oracles (AO).

[Here you can find full description of changes](https://hackmd.io/@lido/HJ0AC5D0Ze).

#### AO CL Balance Decrease

This check prevents an unexpectedly large drop in CL validator balance. It compares the actual balance decrease over up to 36 days with the maximum decrease that could happen naturally from penalties and slashing. The current allowed decrease is capped at about **3.6%** of CL validator balance over the checked period. 

###### First 36 Days Problem
During the first 36 days after the upgrade, there will be no data for the full previous 36-day period.

Effectively, this means that after the release the window starts at 1 day, becomes a 2-day window on day 2, and continues growing until it reaches 36 days.

Even if the oracles were compromised on day 1, this approach still preserves the safety guarantees. A malicious actor could reduce the rebase by 3.6% on day 1, but applying the reduction again on day 2 would result in a cumulative decrease large enough to trigger the sanity check.

#### AO CL Balances Consistency

This is a basic consistency check. It verifies that the sum of validator balances across all staking modules equals the total CL validators balance reported by the oracle. 

#### AO Balance Increase Per Day

This check validates daily balance growth. It ensures that pending ETH, newly activated ETH, validator balance increases, and per-module balance changes stay within expected daily limits, including safety caps for APR and gifts. 

#### AO Exited Validators by Staking Module

This check limits how many validators can be reported as exited per staking module. It was updated to account for consolidation, using a conservative worst-case assumption where validators may have only **16 ETH** balance. This prevents valid withdrawals from being incorrectly rejected. 

#### VEBO Maximum Eth Amount To Exit

This check limits how much ETH can be requested to exit in a single VEBO report. It calculates the total exit amount using the number of validators of each type and their maximum balance.

### Legacy API

Currently, external services use `Lido.getBeaconStat()`, which returns `depositedValidators`, `beaconValidators`, and `beaconBalance`.

However, `beaconValidators` would no longer reflect the actual consensus layer (CL) state and would always be equal to `depositedValidators`.

It is proposed to **mark `getBeaconStat` as deprecated**.

For new integrations, `getBalanceStats()` is recommended, as it reflects the current accounting model and provides more detailed information about the protocol state.

## Consolidation

![Consolidation flow](./assets/lip-35/consolidation_flow.png)

It is suggested that the first step in the consolidation process is an EasyTrack motion to set consolidation parameters.

Operators will use EasyTrack to specify a **source/target operator pair** (CMv1 → CMv2), along with a **Consolidation Manager address**—the address that will be granted permission to submit consolidation requests for consolidating validators from the source operator to the target operator. Once the motion is enacted, stake transfers from a CMv1 operator entity to its corresponding CMv2 entity are permitted, allowing the operator to submit consolidation requests to the Migrator contract.

Once consolidation is allowed, the operator submits key indices to the Consolidation Migrator contract, which verifies that the keys are in use (deposited). The Consolidation Migrator, via dedicated modules, retrieves the corresponding pubkeys from the provided key indices. The consolidation requests are then sent to the Consolidation Bus, which stores the hash of the pubkey batch received from the Migrator.

After the required [execution delay](#execution-delay) has passed, a permissionless executor submits the same batch along with target validator withdrawal credentials (WC) proofs and the required fee. The Consolidation Bus verifies that the submitted batch hash matches the stored value. If it does, the Bus forwards the consolidation request, WC proofs, and fee to the Consolidation Gateway.

The Consolidation Gateway verifies the target validators’ WC proofs, checks consolidation limits, and ensures that deposits in the Deposit Security Module (DSM) are not paused, that Lido is running, and that bunker mode is disabled.

The Gateway then forwards the requests, and the fee to the Withdrawal Vault contract, which submits the request to the system contract.

Dedicated on-chain monitoring would help ensure that operators submit valid consolidation requests (i.e., no attempts to consolidate validators scheduled for exit by VEBO, no consolidation of inactive validators, and no duplicate pending consolidation requests).

### Consolidation EasyTrack

When an operator creates an EasyTrack proposal to request consolidation permission, they would provide:

- **A single source operator ID** in CMv1
- **A list of target operator IDs** in CMv2
- **A single Consolidation Manager address** — the address that will be granted permission to submit consolidation requests for migrating validators from the source operator to all specified target operators

> Note. Operators may specify any Consolidation Manager address. This may be their reward address, if preferred, or any other address of their choosing.

Each proposal defines one source operator and multiple target operators, forming multiple source→target mappings that share the same source. The provided Consolidation Manager will be authorized to execute consolidations from the specified source operator to any of the listed target operators.

Currently, only consolidation from operators in the CMv1 module to operators in the CMv2 module will be allowed. Under the hood, Easy Track validation verifies that the source–target operator pair is registered in the CMv2 meta operators registry, which indicates that the pair is allowed under the migration plan.

A single node operator in the CMv1 module may consolidate into multiple operators in the CMv2 module, and such configurations can be submitted within a single EasyTrack motion.

### Consolidation Migrator

The **Consolidation Migrator** contract validates stake consolidation (migration) requests from a source operator in one module to a target operator in another module. The **source module ID** and **target module ID** are provided at deployment time in the implementation contract and are immutable thereafter.

```
// Consolidation Migrator receive key indexes grouped by target keys
sourceOperatorId
targetOperatorId
groups: [
  {
    sourceKeyIndices: [...], // many sources
    targetKeyIndex           // single target
  },
  ...
]
```

The **Consolidation Migrator** receives a source operator ID, a target operator ID, and key indices grouped by target keys. It then uses the respective modules to resolve the provided key indices into pubkeys and additionally ensures that:

- Consolidation requests are submitted by the authorized **Consolidation Manager address**
- Consolidations are permitted only for explicitly allowed **(sourceOperator → targetOperator)** pairs
- Each validator key involved in a consolidation has been deposited by Lido.

If all checks pass, the **Consolidation Migrator** sends the resolved public keys (grouped by target key) from both the source and target modules to the **Consolidation Bus**.

To manage the list of allowed consolidation pairs (source → target operators), it is proposed to add three methods:

`allowPair` — allows consolidation from a source operator to a target operator for a specified consolidation manager address. This method will be called via an EasyTrack motion upon enactment.

`disallowPair` — disallows consolidation from a source operator to a target operator. This method may be called to correct an incorrectly allowed pair, update the consolidation manager address, or stop consolidation for operational reasons. It is proposed to grant the CMC Committee (also known as CM Committee Multisig) permission to call this method.

`selfDisallowPair` — allows the consolidation manager to revoke (disallow) a previously allowed pair that this manager is responsible for.

Note. Once a consolidation pair is disallowed, it may be allowed again by creating and enacting a new EasyTrack motion by the operator.

```solidity
/// @notice Interface for validating and submitting stake consolidation (migration) requests
///         between operators across two modules.
interface IConsolidationMigrator {
  // =========
  //  Structs
  // =========
  struct ConsolidationIndexGroup {
    uint256[] sourceKeyIndices;
    uint256 targetKeyIndex;
  }

  // =========
  //  Events
  // =========
  event ConsolidationPairAllowed(
    uint256 indexed sourceOperatorId,
    uint256 indexed targetOperatorId,
    address indexed submitter
  );
  event ConsolidationPairDisallowed(
    uint256 indexed sourceOperatorId,
    uint256 indexed targetOperatorId,
    address indexed submitter
  );
  event ConsolidationSubmitted(
    uint256 indexed sourceOperatorId,
    uint256 indexed targetOperatorId,
    ConsolidationIndexGroup[] groups
  );

  // ==================
  //  Read-only views
  // ==================

  /// @notice Gets the source module ID this migrator is bound to.
  function sourceModuleId() external view returns (uint256);

  /// @notice Gets the target module ID this migrator is bound to.
  function targetModuleId() external view returns (uint256);

  /// @notice Returns true if consolidation from `sourceOperatorId` to `targetOperatorId` is allowed.
  function isPairAllowed(uint256 sourceOperatorId, uint256 targetOperatorId) external view returns (bool);

  /// @notice Returns the list of target operators allowed for a given `sourceOperatorId`.
  function getAllowedTargets(uint256 sourceOperatorId) external view returns (uint256[] memory targetOperatorIds);

  /// @notice Returns the submitter address for a consolidation pair.
  function getSubmitter(uint256 sourceOperatorId, uint256 targetOperatorId) external view returns (address);

  /// @notice Returns the StakingRouter address.
  function getStakingRouter() external view returns (address);

  /// @notice Returns the ConsolidationBus address.
  function getConsolidationBus() external view returns (address);

  // ===================
  //    Submission
  // ===================

  /// @notice Submits a batch of consolidation requests after validation.
  /// @dev MUST revert if the batch would fail validation. Emits ConsolidationSubmitted on success.
  /// @dev Caller must be the designated submitter for this pair (set via allowPair).
  function submitConsolidationBatch(
    uint256 sourceOperatorId,
    uint256 targetOperatorId,
    ConsolidationIndexGroup[] calldata groups
  ) external;

  // ======================
  //  Allowlist management
  // ======================

  /// @notice Allows consolidations from `sourceOperatorId` to `targetOperatorId`.
  /// @dev Access-controlled via ALLOW_PAIR_ROLE.
  function allowPair(uint256 sourceOperatorId, uint256 targetOperatorId, address submitter) external;

  /// @notice Disallows consolidations from `sourceOperatorId` to `targetOperatorId`.
  /// @dev Access-controlled via DISALLOW_PAIR_ROLE.
  function disallowPair(uint256 sourceOperatorId, uint256 targetOperatorId) external;

  /// @notice Allows a submitter to disallow their own previously allowed pair.
  /// @dev Permissionless, but restricted to the original submitter of the pair.
  function selfDisallowPair(uint256 sourceOperatorId, uint256 targetOperatorId) external;
}
```

#### Module Interaction

Since the migrator is a temporary contract whose primary purpose is to support the upcoming stake migration from legacy modules to new modules (specifically from CMv1 to CMv2), it is proposed to rely on the existing key-retrieval methods already implemented by these module types.

```solidity
/**
 * @dev Unified interface for staking modules (NOR, SDVT, CMv1, CMv2)
 *      It also works for legacy staking modules (NOR, SDVT) where `getSigningKeys` returns different
 *      tuple `(bytes memory pubkeys, bytes memory signatures, bool[] memory used)`.
 *      The trick: `abi.decode(returndata, (bytes))` will decode only the first tuple element.
 *      This is safe as long as the first returned value really is `bytes pubkeys` in that position.
 */
interface IUnifiedStakingModule {
  function getSigningKeys(
    uint256 nodeOperatorId,
    uint256 startIndex,
    uint256 keysCount
  ) external view returns (bytes memory);

  function getNodeOperatorSummary(
    uint256 _nodeOperatorId
  )
    external
    view
    returns (
      uint256 targetLimitMode,
      uint256 targetValidatorsCount,
      uint256 stuckValidatorsCount,
      uint256 refundedValidatorsCount,
      uint256 stuckPenaltyEndTimestamp,
      uint256 totalExitedValidators,
      uint256 totalDepositedValidators,
      uint256 depositableValidatorsCount
    );
}
```

To validate whether a target key has already been used (deposited), the `getNodeOperatorSummary` method from `IStakingModule` will be used to obtain the total number of deposited validators (`key index < totalDepositedValidators`).

### Consolidation Message Bus

The Message Bus decouples consolidation request submission from execution. This process consists of two steps:

1. Add consolidation requests via the `addConsolidationRequests` method.
2. Execute previously added requests via the `executeConsolidation` method.

Authorized actors (e.g., the Consolidation Migrator) can submit consolidation requests grouped by target key in batches using the `addConsolidationRequests` method:

```
// Add consolidation requests grouped by target key
groups: [
  {
    sourceKeys: [...],   // many sources
    targetKey            // single target
  },
  ...
]
```

The `addConsolidationRequests` method enforces:

- a batch size limit (maximum number of requests), and
- a target validator group limit (maximum number of target validator groups per batch).

It also stores batch hashes along with submission timestamps for deferred execution.

After the execution delay has elapsed, any permissionless executor can process a batch via `executeConsolidation` by submitting:

- the original batch (previously added via `addConsolidationRequests`)
- the required fee
- `ValidatorWitness` proofs for each target validator

```
// Execute consolidation requests
groups: [
  {
    sourceKeys: [...],
    validatorWitness // contains the original target key and withdrawal credentials (WC) proof
  },
  ...
]
```

The `executeConsolidation` verifies that:

- the submitted batch hash matches the stored hash of the original request
- the required execution delay has elapsed.

If all checks pass, the contract forwards the consolidation requests, withdrawal credentials proofs, and fee to the `ConsolidationGateway`. The processed batch is then removed from storage.

The `ConsolidationBus` allows configuring:

- **Batch size limit** via `setBatchSize` — maximum number of requests per batch
- **Target group limit** via `setMaxGroupsInBatch` — maximum number of validator groups per batch
- **Execution delay** via `setExecutionDelay` — minimum time between batch submission and execution

```solidity
/**
 * @title Consolidation Message Bus Interface
 * @notice
 * 1. Admins register/unregister publishers via grant/revoke PUBLISH_ROLE.
 * 2. Registered publishers add consolidation requests (PUBLISH_ROLE).
 * 3. Executor bot executes batches, paying the required ETH fee;
 *    the bus forwards the batch to ConsolidationGateway.
 * 4. Optional REMOVE_ROLE can remove batches from the pending queue.
 */
interface IConsolidationBus {
  // Structs
  struct ConsolidationGroup {
    bytes[] sourcePubkeys;
    bytes targetPubkey;
  }

  struct BatchInfo {
    address publisher;
    uint64 addedAt;
  }

  // Events
  event BatchLimitUpdated(uint256 newLimit);
  event MaxGroupsInBatchUpdated(uint256 newLimit);
  event ExecutionDelayUpdated(uint256 newDelay);
  event RequestsAdded(address indexed publisher, bytes batchData);
  event RequestsExecuted(bytes32 indexed batchHash, uint256 feePaid);
  event BatchesRemoved(bytes32[] batchHashes);

  // View methods
  function batchSize() external view returns (uint256);
  function maxGroupsInBatch() external view returns (uint256);
  function executionDelay() external view returns (uint256);
  function getConsolidationGateway() external view returns (address);
  function getBatchInfo(bytes32 batchHash) external view returns (BatchInfo memory);

  // Role constants
  // bytes32 public constant MANAGE_ROLE = keccak256("MANAGE_ROLE");
  // bytes32 public constant PUBLISH_ROLE = keccak256("PUBLISH_ROLE");
  // bytes32 public constant REMOVE_ROLE = keccak256("REMOVE_ROLE");

  // Admin API (MANAGE_ROLE)
  function setBatchSize(uint256 limit) external;
  function setMaxGroupsInBatch(uint256 limit) external;
  function setExecutionDelay(uint256 delay) external;

  // Remover API (REMOVE_ROLE)
  function removeBatches(bytes32[] calldata batchHashes) external;

  // Publisher API (PUBLISH_ROLE)
  // 1. Verify caller has PUBLISH_ROLE.
  // 2. Verify total batch size and number of groups do not exceed limits.
  // 3. Validate all pubkey lengths are 48 bytes and no source equals its target.
  // 4. Store batch hash and BatchInfo (publisher, addedAt timestamp).
  // 5. Emit RequestsAdded event.
  function addConsolidationRequests(ConsolidationGroup[] calldata groups) external;

  // Executor API
  // 1. Verify the batch was added and not executed or removed.
  // 2. Verify the execution delay has passed since the batch was added.
  // 3. Forward the batch to the ConsolidationGateway with ValidatorWitness proofs.
  // 4. Delete the batch from the pending queue.
  // 5. Emit RequestsExecuted event.
  function executeConsolidation(IConsolidationGateway.ConsolidationWitnessGroup[] calldata groups) external payable;
}
```

#### Execution delay

It is proposed that consolidation request execution can only be processed after an execution delay interval, which can be configured via the `setExecutionDelay` method.

This execution delay ensures that honest Council members have sufficient time to pause the DSM and thereby block the execution of new consolidation requests (the Consolidation Gateway checks that the DSM is not paused) in case keys with invalid withdrawal credentials were deposited.

#### Executor Bot

It is proposed that anyone can permissionlessly execute a consolidation batch by calling `executeConsolidation` method with the same batch which was originally added by authorized actors via `addConsolidationRequests`, along with the required fee and `ValidatorWitness` proofs for each target validator.

A new Consolidation Executor bot is expected to be designed to monitor the Consolidation Bus and execute batches of consolidation requests originally submitted to the Consolidation Message Bus contract once the execution delay has elapsed.

### Consolidation Gateway

It is proposed to introduce a single smart contract entry point for processing consolidation requests. This entry point would be responsible for verifying target validators’ withdrawal credentials, checking consolidation preconditions, enforcing consolidation limits, and enabling the consolidation flow to be paused independently in the event of an emergency.

Using the Withdrawal Vault for this purpose would not be advisable. The Withdrawal Vault manages multiple independent concerns, including protocol withdrawals and triggerable exit requests. Treating it as a pausable unit would prevent selectively pausing consolidation requests while continuing to accept triggerable withdrawal requests, which is operationally undesirable.

To address this, the `ConsolidationGateway` is introduced as a pausable contract (see [PausableUntil.sol](https://github.com/lidofinance/core/blob/master/contracts/0.8.9/utils/PausableUntil.sol)) designed to handle consolidation requests in a controlled and secure manner. It:

- **Authorizes** only permitted callers to submit requests (e.g., the Consolidation Bus)
- **Enforces request limits** to prevent overload
- **Ensures DSM deposits are not paused**
- **Ensures Lido is not stopped and bunker mode is not active**
- **Verifies CL proofs** of target validator withdrawal credentials (via `CLProofVerifier`)
- **Validates ETH fees**, ensuring they meet the minimum consolidation cost
- **Transfers the required fee** to the `WithdrawalVault`
- **Refunds any excess ETH** to a specified recipient
- **Can be paused** via the `CircuitBreaker` mechanism for emergency control

```solidity
interface IConsolidationGateway {
  struct ValidatorWitness {
    bytes32[] proof;
    bytes pubkey;
    uint256 validatorIndex;
    uint64 childBlockTimestamp;
    uint64 slot;
    uint64 proposerIndex;
  }

  struct ConsolidationWitnessGroup {
    bytes[] sourcePubkeys;
    ValidatorWitness targetWitness;
  }

  function addConsolidationRequests(
    ConsolidationWitnessGroup[] calldata groups,
    address refundRecipient
  ) external payable onlyRole(ADD_CONSOLIDATION_REQUEST_ROLE) whenResumed;

  function setConsolidationRequestLimit(
    uint256 maxConsolidationRequestsLimit,
    uint256 consolidationsPerFrame,
    uint256 frameDurationInSec
  ) external onlyRole(EXIT_LIMIT_MANAGER_ROLE);

  function getConsolidationRequestLimitFullInfo()
    external
    view
    returns (
      uint256 maxConsolidationRequestsLimit,
      uint256 consolidationsPerFrame,
      uint256 frameDurationInSec,
      uint256 prevConsolidationRequestsLimit,
      uint256 currentConsolidationRequestsLimit
    );

  function resume() external onlyRole(RESUME_ROLE);
  function pauseFor(uint256 duration) external onlyRole(PAUSE_ROLE);
  function pauseUntil(uint256 pauseUntilInclusive) external onlyRole(PAUSE_ROLE);
}
```

### Withdrawal Vault

A consolidation request is made by signing a transaction with the source validator’s withdrawal address. Since all Lido core validators use **WithdrawalVault** credentials, the consolidation request to the [EIP-7251](https://eips.ethereum.org/EIPS/eip-7251) system contract must be sent from the **WithdrawalVault** contract.

It is proposed to add the following methods to the WithdrawalVault contract:

- `addConsolidationRequests`
- `getConsolidationRequestFee`

Only the **ConsolidationGateway** contract would be permitted to add consolidation requests.

```solidity
interface IWithdrawalVault {
  /**
   * @dev Submits EIP-7251 consolidation requests for each (source, target) pair.
   *      Each request instructs a validator to consolidate its stake to the target validator.
   *
   * @param sourcePubkeys 48-byte public keys of source validators.
   * @param targetPubkeys 48-byte public keys of target validators.
   *
   * @notice Reverts if:
   *         - Caller is not ConsolidationGateway.
   *         - Arrays are empty, malformed, or of unequal length.
   *         - Invalid total withdrawal fee value is provided.
   */
  function addConsolidationRequests(bytes[] calldata sourcePubkeys, bytes[] calldata targetPubkeys) external payable;

  /**
   * @dev Returns the current EIP-7251 consolidation fee per request.
   */
  function getConsolidationRequestFee() external view returns (uint256);
}
```

## Deposits

To fully support `0x02` keys and enable flexible stake rebalancing across operators, it is proposed to introduce **top-up deposits**. With this enhancement, the system will support two types of deposit flows:

- **Predeposits**: deposits of exactly 32 ETH to new validators for `0x01` and `0x02` types of keys
- **Top-ups**: deposits for `0x02`-type active validators intended to reach the max effective balance (2048 ETH)

Following this change, two types of deposit-handling modules may exist:

- Modules with **`0x01` keys** that accept only 32 ETH predeposits
- Modules with **`0x02` keys** that accept predeposits and top-ups up to 2,048 ETH

To maintain system simplicity, **each module must support only one key type** for deposits: either `0x01` or `0x02` — not both.

It is proposed that the deposit flow rely on validator balances rather than the number of keys, and that it support keys with both credential types: `0x01` and `0x02`.

### Depositable ETH Pull Model

The Lido contract is currently tightly coupled to the deposit process. Lido must determine—via the Staking Router—how much of the available ETH should be allocated to each module, and then pass both ETH and the required deposit data into the Staking Router’s `deposit` method. This creates unnecessary complexity in the deposit flow.

![Deposit push model](./assets/lip-35/deposit_push_model.png)

It is proposed to decouple deposit logic from the Lido contract by shifting the deposit flow from a **push** model to a **pull** model. Under this approach, the Staking Router would withdraw ETH from the Lido buffer as needed to execute deposits.

![Deposit pull model](./assets/lip-35/deposit_pull_model.png)

This change would allow:

- Keeping the Lido contract simpler, without deposit-related functions, reducing the number of responsibilities it carries.
- Simplifying the Staking Router logic by making it the single component responsible for executing deposits.
- Implementing all new deposit functionality outside of the Lido contract, in a newer Solidity version.

#### Lido

The Lido contract is responsible for supplying the ETH required for validator top-ups. To support the new ETH pull model during the deposit process, it is proposed to add a new `withdrawDepositableEther` method to the Lido contract.

Only the Staking Router will be permitted to pull ETH from Lido. The `withdrawDepositableEther` method will verify that the requested ETH amount is available for deposits and will then call `StakingRouter.receiveDepositableEther`, attaching the corresponding ETH.

```solidity
interface ILido {
  /// @notice Withdraw depositable ETH, send the requested ETH amount to the StakingRouter.
  /// @dev Can be called only by StakingRouter.
  /// @dev Access-controlled in the implementation (role-based).
  /// @param _amount amount of ETH to withdraw
  /// @param _seedDepositsCount amount of seed deposits. In case of top up this value will be equal to 0
  function withdrawDepositableEther(uint256 _amount, uint256 _seedDepositsCount) external;
}
```

As part of the pull model transition, the following deposit-related members are proposed to be removed from the Lido contract:

- `deposit(uint256 _maxDepositsCount, uint256 _stakingModuleId, bytes _depositCalldata)` — the old push-model deposit method.
- The `IStakingRouter.getStakingModuleMaxDepositsCount()` interface usage — Lido would no longer need to determine per-module deposit allocation, as the Staking Router would handle this independently under the pull model.

#### Staking Router

To support switching the deposit flow from a push model to a pull model, it is proposed to introduce a new `receiveDepositableEther` method for receiving ETH from Lido. The `receiveDepositableEther` method can receive ETH only from Lido.

```solidity
interface IStakingRouter {
  /**
   * @notice A payable function for depositable ether acquisition. Can be called only by `Lido`.
   */
  function receiveDepositableEther() external payable;
}
```

It is proposed that deposit flows adopt the new pull-based ETH model. These flows are covered in detail in the following Deposits sections.

### Predeposit flow

A _predeposit_ refers to the initial 32 ETH deposit made to activate a validator.

For withdrawal credentials of type `0x01`, 32 ETH is both the minimum and the maximum effective balance for activation, so the predeposit constitutes the full deposit.

For withdrawal credentials of type `0x02`, the initial 32 ETH deposit is used to activate the validator, while any additional balance may be deposited later, if applicable.

**Note.** A 32 ETH deposit is suggested because it is the minimum effective balance needed for validator activation. Deposits below this threshold do not activate a validator and therefore do not accrue rewards until the total deposited amount reaches at least 32 ETH.

![Predeposits flow](./assets/lip-35/predeposits_flow.png)

1. **The Depositor Bot** collects guardian signatures and initiates the deposit transaction for the [selected module](#deposit-module-selection).
2. **The DepositSecurityModule** verifies the guardian signatures and checks the deposit distance. If everything is valid, it calls `StakingRouter.deposit`.
3. **The StakingRouter** performs deposits. As part of this process, the Staking Router:
   - Calculates how much of Lido’s available ETH should be allocated to each staking module.
   - Obtains deposit keys from the staking module.
   - Pulls from Lido the total amount of ETH required to execute deposits for the received keys.
   - Submits a 32 ETH deposit for each key.

### Top-up flow

As in the initial predeposit flow, the Staking Router will deposit 32 ETH for both `0x01` and `0x02` keys. To support reaching the maximum effective balance for `0x02` keys, it is proposed to add Merkle-proof–based top-ups via a separate flow. This adds security by reducing trust assumptions and avoids complicating the validator-creation logic. Top-ups are allowed only for modules that support creation of `0x02` keys.

![Top-up flow](./assets/lip-35/topup_flow.png)

1. The Depositor Bot select module and validators inside the module according to the algorithms described in the [Depositor Bot](#depositor-bot) section and calls `TopUpGateway.topUp`, providing staking module data (module ID, key indices, operator IDs), Merkle proofs, and validator consensus-layer (CL) data.
2. `TopUpGateway` verifies the validators’ CL data using Merkle proofs. Based on the verified CL data, it calculates the maximum allowed top-up amount for each validator. It then calls `StakingRouter.topUp`, passing the per-validator top-up limits together with the corresponding validator data (public key, module ID, operator ID, and key index).
3. The `StakingRouter` executes the top-ups:
   - The Staking Router verifies that top-ups are allowed for the module (i.e., the module uses withdrawal credentials of type `0x02`).
   - It determines how much ETH can be allocated to the module.
   - It calls `allocateDeposits` on the staking module with the module’s maximum deposit amount and the keys’ data, and receives the exact top-up amount for each key. The sum of all top-up amounts must be less than or equal to the module’s maximum deposit amount.
   - The Staking Router pulls from Lido the total ETH amount equal to the sum of all top-up amounts returned by the staking module.
   - The Staking Router then executes the top-ups for each key.
   - After all keys have been deposited, the Staking Router performs a sanity check to ensure that the entire amount of ETH requested from Lido has been deposited.

### Depositor Bot

The on-chain [min-first allocation strategy](https://docs.lido.fi/contracts/staking-router/#allocation-algorithm) defines how stake is distributed across modules. The depositor bot can not deposit more stake into any module than allowed by the allocation strategy.

Within the allocation constraint:

- For modules supporting `0x02` keys, the bot performs two types of deposits:
  - **Predeposits** (32 ETH seed deposits per new validator)
  - **Top-ups** (up to 2016 ETH per active validator)
- For modules supporting `0x01` keys, only:
  - **Full deposits** (32 ETH deposits per new 0x01 validator)

For modules that support `0x02 keys`, the system should maintain a sufficient number of validators eligible for top-ups, ensuring that a significant portion of the buffered ETH can be utilized without excessive delays (i.e., without waiting for validators to become active).

The Depositor Bot is proposed to be responsible for maintaining an appropriate balance between predeposits and top-ups in `0x02` modules, with **predeposits prioritized over top-ups**.

To determine module allocations, the Depositor Bot uses the `getDepositAllocations` method from the `StakingRouter` contract.

```solidity
interface IStakingRouter {
    /// @notice Returns new deposits allocation after distributing `_depositAmount`.
    /// @param _depositAmount The maximum ETH amount to allocate.
    /// @param _isTopUp Whether the allocation is for top-ups (true) or predeposits (false).
    /// @return totalAllocated The total amount actually allocated
    /// @return allocated Array of allocated amounts per module
    /// @return newAllocations Array of resulting allocation balances per module
    function getDepositAllocations(uint256 _depositAmount, bool _isTopUp)
        external
        view
        returns (
            uint256 totalAllocated,
            uint256[] memory allocated,
            uint256[] memory newAllocations
        );
}
```

#### Deposit Module Selection

Assume the system contains four modules:

| Module | Type (0x01/0x02) | Balance (ETH) |
| ------ | ---------------- | ------------- |
| A      | 0x02             | 100,000       |
| B      | 0x02             | 10,000        |
| C      | 0x01             | 100,000       |
| D      | 0x01             | 10,000        |

The bot runs every 25 blocks. On each run, the bot executes a **two-stage algorithm**:

1. **Stage 1** — Predeposits to modules with `0x02` keys
2. **Stage 2** — Top-ups for `0x02` modules or full 32 ETH deposits for `0x01` modules

> **Note:** If top-ups are disabled in the Depositor Bot, only full 32 ETH deposits for 0x01 modules are performed.

#### Stage 1. Predeposits to modules with `0x02` keys

At this stage, the bot attempts to perform seed deposits into modules that support `0x02` keys (A and B).

- If deposits are possible, the bot:
  - Selects the module with the **lowest balance** among those with `allocated > 0`
  - Executes the deposits
  - Stops execution

Example:

```
// Modules sorted by balance
// Bot chooses module B (lowest balance with allocation > 0)

Module B:  <- selected
    balance: 10_000
    allocated: 320

Module A:
    balance: 100_000
    allocated: 640
```

- If no allocations are available (e.g., no keys remain or all modules exceed their target share), the bot proceeds to Stage 2.

Example:

```
// No allocation available — skip stage
Module B:
    balance: 10_320
    allocated: 0

Module A:
    balance: 100_640
    allocated: 0
```

#### Stage 2. Top-ups to modules with `0x02` keys or deposits to `0x01` modules

At this stage, the bot considers **top-ups** for `0x02` modules (A and B) or **full deposits** for `0x01` modules (C and D)

- If any allocations are available, the bot:
  - Selects the module with the **lowest balance** among those with `allocated > 0`
  - Executes:
    - a **top-up**, if the module is `0x02`, or
    - a **full deposit**, if the module is `0x01`
  - Stops execution

Example:

```
// Modules sorted by balance
// Bot skips modules with allocated = 0 and selects the lowest eligible one

Module D (0x01):  <- skipped (allocated = 0)
    balance: 10_000
    allocated: 0

Module B (0x02):  <- selected
    balance: 10_320
    allocated: 2000

Module C (0x01):
    balance: 100_000
    allocated: 1600

Module A (0x02):
    balance: 100_640
    allocated: 1000
```

- If no top-ups are allowed for `0x02` modules (e.g., all validators reached MaxEB or allocation is zero), **and** no full deposits are allowed for `0x01` modules (e.g., no available keys or all modules exceed their target share),
  the bot exits.

Example:

```
// No allocation available — bot exits
Module D:
    balance: 10_000
    allocated: 0

Module B:
    balance: 12_320
    allocated: 0

Module A:
    balance: 101_640
    allocated: 0

Module C:
    balance: 101_600
    allocated: 0
```

#### Depositor Bot top-up flow for `0x02` key modules

When the Depositor Bot needs to top up validators in 0x02 modules, it performs the following steps:

- Selects validators based on the module-specific algorithm described in the section below
- Builds Merkle proofs
- Calls `TopUpGateway.topUp` (a new contract that verifies proofs and calls `topUp` on the Staking Router) with the staking module data (module IDs, key indices, operator IDs), proofs, and validator CL data

#### Top-up key selection for CMv2

For **CMv2**, the Depositor Bot operates on a per-operator basis and selects keys based on following algorithm:

1. Based on the allocated amount received from the Staking Router’s `getDepositAllocations` function, the Depositor Bot calls `getDepositsAllocation(uint256 depositAmount)` on the module to determine how much stake should be allocated to each operator.
2. Across this list of operators, the Depositor Bot chooses the oldest validators and checks that:
   - the validator has not exceeded the upper bound for top-ups (`effective_balance + pending_deposits ≤ MAX_EFFECTIVE_BALANCE − TOP_UP_SAFETY_MARGIN − minTopUpGwei`; see the [Top-Up Limit Calculation](#top-up-limit-calculation) section);
   - the validator is active;
   - the validator has not initiated exit on CL (`exitEpoch != FAR_FUTURE`);
   - the validator is not slashed;
   - the validator is not a consolidation target (when source validator is Lido WC);
3. For the selected validators, the Depositor Bot builds Merkle proofs:
   - Proof of the `Validator` container on CL;
4. For the selected validators, the Depositor Bot calculates pending deposit amounts.
5. The Depositor Bot calls the `TopUpGateway` with key data (module ID, key indices, operator IDs), proofs, and validator CL data.

#### Top-up key selection for the future `0x02` version of CSM

For the **`0x02` version of CSM**, the module maintains a global internal queue and exposes a **cursor** that always returns the next validator key that must be processed. Unlike CMv2, key selection is module-wide, not per-operator:

1. Based on the allocated amount received from the Staking Router’s `getDepositAllocations` function, the Depositor Bot queries the module cursor via `getKeysForTopUp` to retrieve the next keys to be topped up.
2. For the selected validators, the Depositor Bot builds Merkle proofs:
   - Proof of the `Validator` container on CL
3. For the selected validators, the Depositor Bot calculates pending deposit amounts.
4. The Depositor Bot calls the `TopUpGateway` with key data (module ID, key indices, operator IDs), proofs, and validator CL data.

Because the `0x02` CSM cursor **should never be blocked**, the Depositor Bot **should not skip any validators**, even if one:

- is slashed,
- is marked for exit,
- has `actual + pending` balance great then max effective balance (2048 ETH),
- or fails any other eligibility condition.

Instead, the module relies on **TopUpGateway** to set a **top-up limit of zero** for such validators and then advance the cursor in the `0x02` version of CSM. This design ensures continuous progress of the CSM queue.

### TopUpGateway

TopUpGateway is a new contract that serves as the entry point for validator top-ups.

It verifies validator status using Merkle proofs. Using this CL data, the TopUpGateway contract computes the maximum permitted top-up amount for each validator.

The contract then forwards these per-validator top-up limits — together with other validator metadata (public key, module ID, operator ID, and key index) — to the StakingRouter contract by calling `StakingRouter.topUp`.

To prevent duplicate keys and incorrect calculation of the top-up amount, TopUpGateway will check that `validatorIndices` does not contain duplicates.

It is proposed that the **`topUp` method be restricted by role**, and that this **role be granted to the Depositor Bot**. Role-based restriction allows protection against potential [over-deposits](#validator-pending-balance) and reduces the [risk](#security-considerations) of imbalances between validators.

```solidity
struct TopUpData {
  uint256 moduleId;
  // Key indices and operator IDs needed to verify the key belongs to the module
  uint256[] keyIndices;
  uint256[] operatorIds;
  uint256[] validatorIndices;
  BeaconRootData beaconRootData;
  ValidatorWitness[] validatorWitness;
  uint256[] pendingBalanceGwei;
}

struct BeaconRootData {
  uint64 childBlockTimestamp; // for EIP-4788 lookup
  uint64 slot; // header slot
  uint64 proposerIndex; // header proposer
}

struct ValidatorWitness {
  // Merkle path: Validator[i] → … → state_root → beacon_block_root
  bytes32[] proofValidator;
  // Validator container fields (except WC)
  bytes pubkey;
  uint64 effectiveBalance;
  bool slashed;
  uint64 activationEligibilityEpoch;
  uint64 activationEpoch;
  uint64 exitEpoch;
  uint64 withdrawableEpoch;
}

interface ITopUpGateway {
  /// @notice Allows topping up Lido Core validators.
  /// @dev Access-controlled in the implementation (role-based).
  function topUp(TopUpData calldata topUps) external;

  function getLastTopUpTimestamp() external view returns (uint256);
  function getMaxValidatorsPerTopUp() external view returns (uint256);
  function getMinBlockDistance() external view returns (uint256);
  function getMaxRootAge() external view returns (uint256);

  function setMaxValidatorsPerTopUp(uint256 newValue) external;
  function setMinBlockDistance(uint256 newValue) external;
  function setMaxRootAge(uint256 newValue) external;

  function canTopUp(uint256 stakingModuleId) external view returns (bool);
}
```

#### Proof building rules

The slot is taken from a selected beacon header, and the timestamp is taken from the execution block that stored the root of this header in the [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788) contract. On-chain verification checks that this timestamp resolves, via EIP-4788, to the corresponding beacon root, and that the Merkle path includes a header node with exactly this `(slot, proposer)`.

The slot used in the proof must not be older than the `_maxRootAgeSec` parameter, which defines the maximum allowed age, in seconds, of the beacon root used to prove the validator, relative to the current timestamp.

This requirement ensures that proofs are generated against recent headers, reducing the probability that the validator exits between proof construction and top-up execution, while still giving the off-chain agent enough time to assemble the proof and the remaining inputs required for the top-up.

Reorganizations are not treated as a separate risk: a proof built on a reorged branch simply fails because the [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788) anchor root is no longer available and the Merkle verification reverts.

TopUpGateway enforces that the block timestamp used in the proof is newer than the timestamp of the last top-up.

To perform a top-up for a given validator, an off-chain bot must provide all [consensus-layer validator fields](https://github.com/ethereum/consensus-specs/blob/2b83d5a2/specs/phase0/beacon-chain.md?plain=1#L387-L399), except for the withdrawal credentials, which are obtained from the Staking Router.

The Merkle proof for the validator must establish that the hash-tree-root of this validator container (with the expected withdrawal credentials and other fields) is a leaf of the beacon state tree whose root is the state root committed by the proved beacon header. In other words, the validator record is proven to belong to the state corresponding to the header whose root is exposed via [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788) for the supplied timestamp.

#### Validator State Validation

After verifying the validator state via proofs, the `TopUpGateway` validates the following:

- **Withdrawal credentials** should match the expected `0x02`-format Lido withdrawal credentials.
- **Activation status:** the validator should be activated before the proved header slot:
  - `activationEpoch` should be less than the epoch corresponding to the proved slot.

If any of these checks fail, `TopUpGateway` reverts.

**Note.** If `exitEpoch != FAR_FUTURE_EPOCH` (i.e., an exit has been scheduled or the validator is slashed), the top-up limit is set to `0` (see [Top-Up Limit Calculation](#top-up-limit-calculation)). Therefore, there is no requirement to restrict inputs to validators with an unknown exit epoch or an unslashed status.

#### Validator Pending Balance

To keep the system practical and avoid prohibitive proof sizes, the proposed design does not require the TopUpGateway to verify pending deposits via proofs.

Since top-ups are intended to be a permissioned operation, the `TopUpGateway` relies on an authorized Depositor Bot to provide the correct pending balance.

If pending deposits exist in the queue but are not supplied to the TopUpGateway, a validator may temporarily receive more ETH than necessary. Any excess will eventually be skimmed to the validator’s withdrawal credentials.

#### Validator Top-Up Limit Calculation

Based on a validator’s current status and the amount of pending deposits, the `TopUpGateway` contract computes the maximum permitted top-up amount for each validator.

If a validator is slashed, marked for exit, or has already exited, the allowed top-up amount is set to zero:

```
Top-up limit = 0
```

If the validator is eligible for deposits, the top-up limit is the gap between the validator's current committed balance and a configured ceiling:

```
raw_top_up = targetBalanceGwei − effective_balance − pendingBalanceGwei

Top-up limit = raw_top_up,
               if raw_top_up value is at least minTopUpGwei

Top-up limit = 0,
               if raw_top_up value is less than minTopUpGwei
```

Two configurable parameters control the calculation:

- **targetBalanceGwei** — the validator balance ceiling after top-up. Initialized to `MAX_EFFECTIVE_BALANCE − TOP_UP_SAFETY_MARGIN = 2046.75 ETH`.
- **minTopUpGwei** — the minimum per-validator top-up amount. Initialized to **2 ETH**. If the calculated top-up falls below this threshold, the limit is set to **0**.

For simplicity, it was suggested to use `effective_balance` instead of `active_balance` in the top-up limit calculation. A validator’s `active_balance` may differ from its `effective_balance` by no more than the hysteresis thresholds defined in the consensus layer:

- `DOWNWARD_THRESHOLD = 0.25 ETH`
- `UPWARD_THRESHOLD = 1.25 ETH`

To avoid over-depositing a validator, `TOP_UP_SAFETY_MARGIN` is introduced. It is therefore proposed to set `TOP_UP_SAFETY_MARGIN` to **1.25 ETH**.

This guarantees that, under the current hysteresis thresholds, topping up a validator to the limit will result in a balance between `MAX_EFFECTIVE_BALANCE − 1.5 ETH` and `MAX_EFFECTIVE_BALANCE` (for simplicity, excluding any balance changes during the deposit queue wait time). Under current conditions, it is expected to take up to 12 days for the validator to earn 1.5 ETH and reach `MAX_EFFECTIVE_BALANCE`.

Revisiting the upper bound on `effective_balance + pending_deposits` with `TOP_UP_SAFETY_MARGIN = 1.25 ETH`, we get:

```
effective_balance + pending_deposits ≤ MAX_EFFECTIVE_BALANCE − TOP_UP_SAFETY_MARGIN − minTopUpGwei
```

#### Top-Up Limits

It is proposed to limit the overall amount of ETH that may be topped up into the modules using the following parameters:

- **maxTopUpPerBlockGwei** — the maximum amount of ETH that can be topped up within a single block.
- **minTopUpBlockDistance** — the minimum number of blocks required between `topUp` calls.

This design assumes that **minTopUpBlockDistance** and **maxTopUpPerBlockGwei** are common values for all modules, meaning they define protocol-wide limits rather than per-module top-up limits.

#### Top-up Precondition and Pause

It is proposed that top-ups should not be executed if the protocol is in bunker mode or paused, or if the module is not active.

If DSM paused, it is not risky for top-ups, since during top-ups the withdrawal credentials are verified on-chain.

Additionally it is proposed that TopUpGateway contract **can be paused via the `CircuitBreaker`** mechanism to provide emergency control.

If it becomes necessary to temporarily stop top-ups (e.g., during a hard fork), this can be achieved operationally. Since top-ups are permissioned (i.e., only the Depositor Bot is authorized), the bot can simply be stopped as a precautionary measure.

### Staking Router

It is proposed to update the existing `deposit` methods to support the ETH pull model and enable initial 32 ETH deposits to `0x02` withdrawal credentials. The updated `deposit` method would:

1. **Calculate module allocation.** The amount of ETH that can be deposited into a module is divided by 32 (for both module types) to determine the maximum number of 32 ETH deposits (`maxDepositsCount`). This value is additionally capped by `maxDepositsCountPerBlock`, which can be configured individually for each module.
2. **Obtain keys for the 32 ETH deposit.** After determining the deposit limit, the Staking Router calls the existing `IStakingModule(stakingModuleAddress).obtainDepositData(maxDepositsCount, depositCalldata)` method to fetch public keys and signatures. The `obtainDepositData` call may return up to `maxDepositsCount` keys.
3. **Pull the required ETH from Lido.** The Staking Router pulls from Lido the total amount of ETH required to execute the deposits for the keys obtained from the module. The required amount is calculated as: `number of obtained keys × 32 ETH`.
4. **Perform 32 ETH deposits.** The Staking Router performs a 32 ETH deposit for each key, using the appropriate withdrawal credentials based on `withdrawalCredentialsType`:
   - `0x02` + withdrawal credentials contract address
   - `0x01` + withdrawal credentials contract address
5. **Sanity checks.** After all keys have been deposited, the Staking Router performs a sanity check to ensure that the entire amount of ETH requested from Lido has been deposited.

To support the new top-up flow, it is proposed to add a new `topUp` method to perform validator top-ups. The `topUp` method will:

1. **Check module eligibility for top-ups.** Verifies that the module supports top-ups (i.e., it uses withdrawal credentials of type `0x02`). If the module is not eligible, the call reverts.
2. **Calculate module allocation.** Determines the maximum total amount of ETH that can be allocated to the module according to the MinFirst allocation strategy. To compute each module's current stake, the Staking Router queries `0x02` modules directly via `getTotalModuleStake()`, while for `0x01` modules the value `(depositedValidators − exitedValidators) × 32 ETH` is used based on on-chain accounting. The allocation is expressed in 32 ETH units, so the amount allocated to the module is always a multiple of 32 ETH.
3. **Obtain top-up amounts.** Calls `allocateDeposits` on the staking module, passing the module’s maximum deposit amount along with key indices, operator IDs, public keys, and per-key top-up limits. The `allocateDeposits` method returns the computed top-up amounts for the supplied public keys.
4. **Validate the returned total top-up amount.**
   - Verifies that each returned top-up amount does not exceed the corresponding per-key top-up limit.
   - Verifies that the sum of all returned per-key top-up amounts is less than or equal to the module’s maximum allocated top-up amount for the call.
5. **Pull the required ETH from Lido.** Pulls from Lido the total amount of ETH required to execute the top-ups for the keys returned by the module. The required amount equals the sum of the returned per-key top-up amounts.
6. **Execute top-ups.** Executes a top-up for each returned key using the corresponding amount provided by the module.
7. **Sanity checks.** After all top-ups are executed, performs a sanity check to ensure that the entire amount of ETH requested from Lido has been deposited.

```solidity
interface IStakingRouter {
  /// @notice Invokes a deposit call to the official Deposit contract.
  /// @param _stakingModuleId Id of the staking module to be deposited.
  /// @param _depositCalldata Staking module calldata.
  /// @dev Only the DepositSecurityModule is allowed to call this method.
  function deposit(uint256 _stakingModuleId, bytes calldata _depositCalldata) external;

  /// @notice Method performs top-up calls to the official Deposit contract. Determines how much Lido buffered ether can be deposited
  /// to the staking module, obtains keys from the staking module with exact allocation for each key, pulls ether from Lido,
  /// and performs the top-up call.
  /// @param _stakingModuleId Id of the staking module to be deposited.
  /// @param _keyIndices List of keys' indices
  /// @param _operatorIds List of operator indices
  /// @param _pubkeys List of public keys
  /// @param _topUpLimits Maximum amount (in wei) that can be deposited per key based on CL data and TopUpGateway logic.
  /// @dev Only the TopUpGateway is allowed to call this method.
  function topUp(
    uint256 _stakingModuleId,
    uint256[] calldata _keyIndices,
    uint256[] calldata _operatorIds,
    bytes[] calldata _pubkeys,
    uint256[] calldata _topUpLimits
  ) external;
}
```

In addition to the deposit flow, the `IStakingRouter` interface is extended with the following groups of methods needed to support the new `0x02` module type and the on-chain balance accounting introduced by this proposal:

- **Fee management** — `updateAllStakingModulesFees` performs an atomic batch update of staking module and treasury fees, ensuring that fee changes across all modules take effect in a single transaction and remain consistent (for all modules `staking module fees + treasury fees` is equal) .
- **Validator balances reporting** — `reportValidatorBalancesByStakingModule` is the state-mutating entry point used by the Accounting Oracle to deliver per-module validator balance data to the StakingRouter, where it is stored as each module's `validatorsBalanceGwei` and used as the basis for rewards distribution. `validateReportValidatorBalancesByStakingModule` is its view-only counterpart, used by the sanity checker and the oracle to pre-validate a report against the current module set and configured limits before it is submitted on-chain.
- **Validators balance accessors** — `getModuleValidatorsBalance` and `getTotalModulesValidatorsBalance` return the active validators balance used for rewards distribution, either for a single module or aggregated across all registered modules. 
- **Module state getters** — `getStakingModuleStateConfig`, `getStakingModuleStateDeposits`, and `getStakingModuleStateAccounting` expose a module's state split into three logical parts (configuration, deposit-related fields, and accounting) to keep the returned structures small and decoupled, allowing consumers to fetch only the slice they need.
- **Module-level helpers** — `getStakingModuleWithdrawalCredentials` returns the per-module withdrawal credentials with the module's type prefix applied (`0x01...` or `0x02...`), and `canDeposit` reports whether a module exists and is currently eligible to receive deposits.
- **Versioning** — `getContractVersion` returns the current initialized version of the proxy, replacing the previous `Versioned` helper.

```solidity
interface IStakingRouter {
  /// @notice Updates fees for all staking modules in a single atomic operation.
  /// @param _stakingModuleFees New staking module fee values in the current module iteration order (returned by `getStakingModuleIds()`).
  /// @param _treasuryFees New treasury fee values in the current module iteration order.
  /// @dev The function is restricted to the `STAKING_MODULE_MANAGE_ROLE` role.
  function updateAllStakingModulesFees(
    uint256[] calldata _stakingModuleFees,
    uint256[] calldata _treasuryFees
  ) external;

  /// @notice Submits per-module validator balances to the StakingRouter.
  /// @param _stakingModuleIds Ids of the staking modules included in the report, in the current iteration order.
  /// @param _validatorBalancesGwei Total CL balance attributed to each module's active validators (gwei), aligned with `_stakingModuleIds`.
  /// @dev Called by the Accounting Oracle as part of the main report phase. The reported values are stored as each
  ///      module's `validatorsBalanceGwei` and used by the rewards distribution logic.
  function reportValidatorBalancesByStakingModule(
    uint256[] calldata _stakingModuleIds,
    uint256[] calldata _validatorBalancesGwei
  ) external;

  /// @notice Validates a validator balances report against the current StakingRouter module set and limits.
  /// @dev View-only counterpart of `reportValidatorBalancesByStakingModule` used by the sanity checker / oracle to
  ///      pre-validate report data without mutating state.
  function validateReportValidatorBalancesByStakingModule(
    uint256[] calldata _stakingModuleIds,
    uint256[] calldata _validatorBalancesGwei
  ) external view;

  /// @notice Returns the configuration part of a staking module's state
  ///         (module address, fees, share limits, status, withdrawal credentials type).
  function getStakingModuleStateConfig(uint256 _stakingModuleId)
    external
    view
    returns (ModuleStateConfig memory stateConfig);

  /// @notice Returns the deposits-related part of a staking module's state
  ///         (last deposit timestamp/block, max deposits per block, min deposit block distance).
  function getStakingModuleStateDeposits(uint256 _stakingModuleId)
    external
    view
    returns (ModuleStateDeposits memory stateDeposits);

  /// @notice Returns the accounting part of a staking module's state.
  /// @return validatorsBalanceGwei Total CL balance attributed to the module's active validators (gwei).
  /// @return exitedValidatorsCount Total exited validators count tracked by the StakingRouter for the module.
  function getStakingModuleStateAccounting(uint256 _stakingModuleId)
    external
    view
    returns (uint64 validatorsBalanceGwei, uint64 exitedValidatorsCount);

  /// @notice Returns the per-module withdrawal credentials with the module's type prefix applied.
  /// @return 0x01... for legacy (`0x01`) modules, 0x02... for new (`0x02`) modules.
  function getStakingModuleWithdrawalCredentials(uint256 _stakingModuleId) external view returns (bytes32);

  /// @notice Returns whether the staking module exists and is in the `Active` status, i.e. eligible to receive deposits.
  function canDeposit(uint256 _stakingModuleId) external view returns (bool);

  /// @notice Returns the staking module's active validators balance used for rewards distribution.
  /// @dev For `0x01` modules this is derived from on-chain accounting; for `0x02` modules it is the module's
  ///      `validatorsBalanceGwei` tracked by the StakingRouter and updated by the oracle.
  function getModuleValidatorsBalance(uint256 _stakingModuleId) external view returns (uint256);

  /// @notice Returns the sum of `getModuleValidatorsBalance` across all registered staking modules.
  function getTotalModulesValidatorsBalance() external view returns (uint256);

  /// @notice Returns the current initialized version of the proxy (replaces the previous `Versioned` helper).
  function getContractVersion() external view returns (uint256);
}
```

#### Staking Router Configuration

It is proposed to add a new `withdrawalCredentialsType` property to the module configuration in the `StakingRouter`. This property would distinguish modules that use `0x01` withdrawal credentials from modules that use `0x02` withdrawal credentials and support top-ups up to 2,048 ETH.

```solidity
struct StakingModule {
    ...

    /// @notice The type of withdrawal credentials used for validator creation.
    uint8 withdrawalCredentialsType;
}
```

#### Request limits

As the `topUp` method on `TopUpGateway` will be restricted by role, it is sufficient to limit requests based on top-up block distance, as is done in the DSM.

#### Module stake allocation

The Staking Router distributes incoming ETH across staking modules using the MinFirst allocation strategy: modules with proportionally less stake receive deposits first, gradually equalizing their sizes over time.

Each module's capacity is capped by two constraints: its number of depositable validator keys, and a configured stake share limit. The effective capacity is the minimum of the two:

**Initial deposits:**

```
capacity = min(shareLimit × totalValidators / BASIS_POINTS, currentAllocation + depositableCount)
```

**Top-up deposits** follow the same priority ordering, but a `0x02` type module's capacity is measured by the remaining effective balance headroom of its active validators rather than available keys:

```
capacity = min(shareLimit × totalValidators / BASIS_POINTS, activeCount × maxEBType2 / maxEBType1)
```

The allocation strategy operates in `maxEBType1` (32 ETH) units, so the amount allocated to each module is always a multiple of 32 ETH.

For `0x01` modules, the Staking Router derives the current allocation from on-chain accounting as the number of active validators (each holding 32 ETH):

```
currentAllocation = depositedValidators - max(moduleReportedExited, stakingRouterTrackedExited)
```

For `0x02` modules, the Staking Router queries the module directly via `getTotalModuleStake()` to obtain the actual total ETH staked in the module, then converts it into 32 ETH units (rounded up):

```
currentAllocation = ceil(getTotalModuleStake() / maxEBType1)
```

### Staking Modules

It is proposed to add a new `IStakingModuleV2` interface, which will include the methods required by the new deposit flow:

- `allocateDeposits` — method to obtain from the module the top-up amounts for the provided public keys. The module also verifies that the keys belong to this module and reverts if invalid data is provided.
- `getTotalModuleStake` - method to obtain the total amount of ETH staked in the module

```solidity
interface IStakingModuleV2 {
  /// @notice Validates provided keys and calculates deposit allocations for top-ups.
  /// @dev Reverts if any key doesn't belong to the module or data is invalid.
  /// @param maxDepositAmount Total ether amount available for top-ups (must be a multiple of 1 gwei).
  /// @param pubkeys List of validator public keys to top up.
  /// @param keyIndices Indices of keys within their respective operators.
  /// @param operatorIds Node operator IDs that own the keys.
  /// @param topUpLimits Maximum amount that can be deposited per key based on Consensus Layer data and SR internal logic.
  /// @return allocations Amount to deposit to each key.
  /// @dev Values maxDepositAmount, topUpLimits, allocations are denominated in wei.
  /// @dev allocations can contain zero values.
  /// @dev sum(allocations) must be less than or equal to maxDepositAmount.
  function allocateDeposits(
    uint256 depositAmount,
    bytes[] calldata pubkeys,
    uint256[] calldata keyIndices,
    uint256[] calldata operatorIds,
    uint256[] calldata topUpLimits
  ) external returns (uint256[] memory allocations);

  /// @notice returns the total amount of ETH staked in the module, in wei.
  function getTotalModuleStake() external view returns (uint256);
}
```

#### CMv2

The new Curated module is expected to include this method to guide the Depositor Bot on which operators need to be topped up.

```solidity
/// @notice Returns operators and the amount of ETH that can be allocated to each operator from `depositAmount`.
/// @param depositAmount Amount of ETH that can be deposited to the module.
function getDepositsAllocation(
  uint256 depositAmount
) external view returns (uint256 allocated, uint256[] memory operatorIds, uint256[] memory allocations);
```

#### `0x02` version of CSM

In the `0x02` version of the CSM module, it is proposed to include this method to provide the Depositor Bot with the ordered list of validator keys that need to be topped up next.

```solidity
/// @notice Fetches up to `keysCount` validator public keys from the front of the top-up queue.
/// @dev If the queue contains fewer than `keysCount` entries, all available keys are returned.
/// @dev The keys are returned in the same order as they appear in the queue.
/// @param keysCount The maximum number of keys to retrieve.
/// @return pubkeys The list of validator public keys returned from the queue.
function getKeysForTopUp(uint256 keysCount) external view returns (bytes[] memory pubkeys);
```

## Validators Exits

It is proposed to update the Validators Exit flow:

- **On-chain:** Update the exit report format and sanity checks to support `0x02` keys by validating exits using an upper-bound total effective balance (key-type aware) instead of validator-count limits.
- **Off-chain:** Update the exit selection logic to support deposit reserve buffer effects, consolidation-aware exits, balance-based and operator-weight–based withdrawals.

### Validators Exit Bus Oracle (VEBO)

Currently, the `ValidatorsExitBusOracle.sol` contract performs a sanity check on the number of exiting validators included in a VEBO report, under the assumption that every validator uses a `0x01`-type key with a maximum effective balance of 32 ETH.

After the transition to MaxEB, the validator count–based approach in VEBO is no longer sufficient: a validator’s effective balance may range from 32 ETH up to 2048 ETH, so the number of validators alone does not provide a reliable upper bound on the withdrawn amount.

To ensure a reliable upper bound on the withdrawn amount, it is proposed to:

- Extend the VEBO report format to include the pubkey, module ID, operator ID, and key index, allowing the key type (`0x01` / `0x02`) to be determined on-chain.
- Update the sanity checker to validate the **upper-bound total effective balance** requested to exit based on the key type (`0x01` with a MaxEB of 32 ETH or `0x02` with a MaxEB of 2048 ETH), rather than the raw validator count.

![VEBO flow](./assets/lip-35/vebo_flow.png)

#### Report format

It is proposed to introduce a second VEBO report data format, `DATA_FORMAT_LIST_WITH_KEY_INDEX`, which will include the pubkey, module ID, operator ID, and key index, allowing the key type (`0x01` / `0x02`) to be determined on-chain.

```
/// Current DATA_FORMAT_LIST = 1
/// MSB <------------------------------------------------------- LSB
/// |  3 bytes   |  5 bytes   |     8 bytes      |    48 bytes     |
/// |  moduleId  |  nodeOpId  |  validatorIndex  | validatorPubkey |

/// NEW DATA_FORMAT_LIST_WITH_KEY_INDEX = 2
/// MSB <-------------------------------------------------------------------- LSB
/// |  3 bytes   |  5 bytes   |     8 bytes      |   8 bytes  |    48 bytes     |
/// |  moduleId  |  nodeOpId  |  validatorIndex  |  keyIndex  | validatorPubkey |
```

It is proposed that oracles submit exit reports using the new `DATA_FORMAT_LIST_WITH_KEY_INDEX` format, enabling the key type (`0x01` / `0x02`) to be determined reliably on-chain.

At the same time, existing trusted entities (i.e., Easy Track for the Curated and SDVT modules) are expected to continue operating with the current `DATA_FORMAT_LIST` format without changes. This is acceptable because Easy Track already performs an on-chain verification that the reported keys belong to the corresponding module before submitting exit requests.

#### Sanity checker

The proposed update is to validate the **upper-bound total effective balance** requested to exit rather than the raw validator count. In `OracleReportSanityChecker.sol`:

- The existing `maxValidatorExitRequestsPerReport` limit will be replaced with a balance-based limit, `maxBalanceExitRequestedPerReportInEth`.
- To preserve the current safety threshold, `maxBalanceExitRequestedPerReportInEth` should be set to **19,200 ETH**, equivalent to the previous cap of `600 validators × 32 ETH`.

Under the new logic, each VEBO report will be validated as follows:

- For modules with **`0x01`-type keys**, multiply the number of validators in the report that belong to this module by **32 ETH**.
- For modules with **`0x02`-type keys**, multiply the number of validators in the report that belong to this module by **2048 ETH**.
- Sum the resulting values across all modules and compare the total against the configured **ETH-denominated sanity-check limit**.

This approach establishes a conservative upper bound on withdrawal volume per report and ensures that VEBO cannot trigger an excessively large withdrawal in a single submission, thereby reducing protocol risk.

### Off-chain Validators Exit Oracle

It is proposed to update the Validators Exit Oracle off-chain logic to correctly support:

- Deposit reserve
- Consolidation
- Operator weight in the CMv2 module

#### Deposit reserve

When Validators Exit Oracle decides how many validators need to be exited, it also looks at the amount of ETH in the buffer.

Currently, Validators Exit Oracle assumes that all of this amount can be used to fulfill Withdrawal Requests and therefore reduces the number of validator exits, assuming that the buffer ETH will cover part of the requests.

However, if deposit reserve is introduced, part of the buffer may be used for deposits that bypass withdrawals.

Therefore, Validators Exit Oracle should take deposit reserve into account when calculating the amount of ETH that needs to be withdrawn.

#### Consolidation

During stake migration from CMv1 to CMv2, and in other cases after the migration is finished:

- Source consolidation validators **should not** be selected for exit by Validators Exit Oracle; otherwise, exit requests would be wasted on validators that are already about to be consolidated.
- Target validators **can** be selected by Validators Exit Oracle, but it should take into account the balances of their associated consolidation source validators, since those balances will be consolidated into the target validator.

It is expected that VEBO would inspect the `PendingConsolidation` queue to differentiate exit requests from consolidation requests when calculating the required ETH withdrawal amount.

#### Operator weight in the CMv2 module

For withdrawals, Validators Exit Oracle must be able to determine from which modules and from which operators stake should be withdrawn.

At the **module level**, Validators Exit Oracle continues to rely on the existing exit share limit.

At the **operator level**, the oracle uses a single weight-proportional target-deviation formula across all modules (the fourth predicate in the exit-prioritization algorithm below). The formula is balance-based — replacing the pre-LIP-35 count-based prioritization, which assumed every validator equals 32 ETH and would mis-rank operators once `0x02` validators with effective balances up to 2048 ETH come online. The same formula produces different effective behaviors depending only on how each module assigns operator weights — explicit on-chain weights for CMv2, off-chain aggregation via the Meta Registry for CMv1, and a uniform baseline weight for CSM, SDVT, and unreferenced CMv1 operators. The per-module behavior is detailed below the predicate table.

To support the explicit-weight case, it is proposed to add a `getOperatorWeights` method to the module interface; only CMv2 implements it.

```solidity
interface ITargetAllocation {
  function getOperatorWeights(uint256[] calldata operatorIds) external view returns (uint256[] memory operatorWeights);
}
```

**CMv1 operator weights via the Meta Registry.** CMv1 operator weights are not exposed on-chain; the Validators Exit Oracle computes them off-chain by aggregating CMv2 weights according to the Meta Registry mapping. This aggregation is necessary because the stake migration from CMv1 to CMv2 occurs unevenly — some operators migrate earlier than others — so the oracle must account for aggregate operator stakes and weights across both modules to determine from which CMv1 operators stake exits should be initiated.

The [Meta Registry](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-33.md#meta-operators-registry) stores the explicit relationship between CMv1 module operators and their corresponding operators in the CMv2 module, expressed as groups in which a CMv1 operator may be listed as an external operator alongside one or more CMv2 sub-operators.

When selecting validators to exit from the CMv1 module, the Validators Exit Oracle uses this mapping to derive each CMv1 operator's aggregate stake and weight:

- **Aggregate stake**: a CMv1 operator's `currentStake` plus the `currentStake` of its linked CMv2 sub-operators. When a CMv2 group lists multiple external operators, that group's sub-operator stake is split equally between them.
- **Aggregate weight**: the sum of `operatorWeight` values from the CMv1 operator's linked CMv2 sub-operators. When a CMv2 group lists multiple external operators, that group's sub-operator weight is split equally between them. CMv1 operators that are not referenced in any CMv2 group fall back to a baseline weight (`100_000`).

The Validators Exit Oracle then computes a target stake per CMv1 operator using the same `(totalStake × operatorWeight) / totalWeight` formula as for CMv2, with `totalStake` and `totalWeight` summed across all CMv1 operators after the aggregation above. This ensures that exits from CMv1 follow the same ValMart-style distribution as CMv2, treating an operator's already-migrated and not-yet-migrated stake as a single allocation.

**Exit prioritization predicates.** To determine which validators to request for exit, Validators Exit Oracle builds a sorted list of exitable validators based on the predicates described in the table below. It then selects entries from this list until the withdrawal queue (WQ) demand is covered by the exiting validators and future rewards, or until the per-report limit is reached.

The fourth predicate is the one introduced by this proposal; the other four are unchanged from the current Validators Exit Oracle implementation.

_The full list of predicates used by Validators Exit Oracle to build the sorted list of exitable validators:_

| Module                                      | Node Operator                                                                  | Validator              |
| ------------------------------------------- | ------------------------------------------------------------------------------ | ---------------------- |
|                                             | Highest number of targeted validators for boosted exit                         |                        |
|                                             | Highest number of targeted validators for smooth exit                          |                        |
| Highest deviation from the exit share limit |                                                                                |                        |
|                                             | **Highest deviation of operator's current stake from its weight-proportional target stake** |                        |
|                                             |                                                                                | Lowest validator index |

The fourth predicate is a single unified formula across all modules: `currentStake − totalModuleStake × operatorWeight / totalModuleWeight`. The effective behavior depends on how operator weights are populated:

- For **CSM** and **SDVT**, all operators share the same default baseline weight (`100_000`), so the target stake reduces to the per-operator average and the predicate ranks operators by current balance descending (i.e., "largest operator first").
- For **CMv2**, operator weights are read on-chain via `getOperatorWeights`; the predicate ranks operators by how much their current stake exceeds their weight-proportional share.
- For **CMv1**, operators referenced in CMv2 groups inherit aggregated weights via the Meta Registry (as described above); unreferenced CMv1 operators fall back to the baseline weight.

## Stake Rebalancing

The current sporadic stake rebalancing between modules via new deposits and withdrawals raises several challenges that are difficult to solve efficiently with existing rebalancing approaches:

- Efficiently rebalance stake between modules; it took SDVT over a year and a half to reach its target share. A future possible CSM share limit increase up to 10% might require significant time.
- Ensure that there is always enough ETH for initial 32 ETH deposits to `0x02`-type keys in the CMv2 module to support stake migration from the CMv1 to the CMv2 module via consolidation.

To solve these challenges, it is proposed to add a deposit reserve mechanism to protect a portion of the protocol's buffered ether for CL deposits, regardless of withdrawal demand.

### Deposit Reserve

Under current conditions, cycling arbitrage and vampire attacks via withdrawals can result in submitted ether being withdrawn before it is ever deposited. This situation limits the ability to allocate stake to new modules and node operators.

To ensure that there is always enough ETH for stake rebalancing and initial 32 ETH deposits to `0x02`-type keys in the CMv2 module during the migration process, it is proposed to reserve a portion of the protocol's buffered ether for CL deposits, protecting it from being consumed by withdrawal demand.

#### Proposed solution

1. Allow the DAO or a delegated entity to set a deposits reserve target (`depositsReserveTarget`) — the amount of buffered ether to protect for CL deposits between accounting oracle reports.
2. On each oracle report, the effective deposits reserve is synced (restored) to the configured target, up to the available buffered ether.
3. The deposits reserve is filled with the highest priority: buffered ether is first allocated to the deposits reserve, then to covering withdrawal requests, and only after that the remainder becomes available for additional CL deposits.

The mechanism introduces two storage values in the Lido main contract:

1. **`depositsReserve`** — the current effective reserve amount, consumed as CL deposits are performed and restored on each oracle report.
2. **`depositsReserveTarget`** — the governance-configured target that `depositsReserve` is restored to on each oracle report.

And affects the following functions:

1. `Lido.getDepositableEther()` — returns total depositable ether (deposits reserve + unreserved).
2. `Lido.withdrawDepositableEther()` — spends depositable buffer and decreases stored deposits reserve accordingly. See [Depositable ETH Pull Model](#depositable-eth-pull-model) for more details.

A setter function is presented, governed by the DAO via `BUFFER_RESERVE_MANAGER_ROLE`:

```solidity
interface ILido {
  /// @notice Set deposits reserve target.
  /// @dev Access-controlled in the implementation (BUFFER_RESERVE_MANAGER_ROLE).
  function setDepositsReserveTarget(uint256 _newDepositsReserveTarget) external;

  /// @notice Returns configured target for deposits reserve.
  function getDepositsReserveTarget() external view returns (uint256);

  /// @notice Returns the currently effective deposits reserve — buffer portion
  ///         available for CL deposits, protected from withdrawals demand.
  /// @dev Capped by current buffered ether.
  function getDepositsReserve() external view returns (uint256);

  /// @notice Returns the currently effective withdrawals reserve.
  /// @dev Computed after deposits reserve is applied.
  function getWithdrawalsReserve() external view returns (uint256);
}
```

#### Buffered Ether Allocation

Buffered ether is split into three priority-ordered buckets:

1. **Deposits Reserve** — per-frame CL deposit allowance, filled first from the buffer.
2. **Withdrawals Reserve** — covers unfinalized withdrawal requests from the remaining buffer.
3. **Unreserved** — excess buffer available for additional CL deposits beyond the reserve.

```
┌─────────── Total Buffered Ether ───────────┐
├────────────────────┬───────────────────────┼─────┬──────────────┐
│●●●●●●●●●●●●●●●●●●●●│●●●●●●●●●●●●●●●●●●●●●●●●○○○○○│○○○○○○○○○○○○○○│
├────────────────────┼───────────────────────┼─────┼──────────────┤
└─ Deposits Reserve ─┼─ Withdrawals Reserve ─┘     ├─ Unreserved ─┘
                     └───── Unfinalized stETH ─────┘

● — covered by Buffered Ether
○ — not covered by Buffered Ether
```

The allocation is computed as follows:

```
depositsReserve    = min(totalBuffered, storedDepositsReserve)
withdrawalsReserve = min(totalBuffered - depositsReserve, unfinalizedStETH)
unreserved         = totalBuffered - depositsReserve - withdrawalsReserve
```

The depositable ether is the sum of deposits reserve and unreserved:

```
depositableEther = depositsReserve + unreserved = totalBuffered - withdrawalsReserve
```

**Example 1:** An ETH amount not exceeding the deposits reserve target is available in buffer. In this case, the deposits reserve absorbs the ETH, protecting it for CL deposits. Withdrawal requests should be covered by validator exits via VEBO.

![Deposits reserve example 1](./assets/lip-35/deposits_reserve_example_1.png)

**Example 2:** An ETH amount greater than the deposits reserve target but insufficient to cover all withdrawal requests is available in buffer. In this case, the deposits reserve is filled first, and the remaining ETH in the buffer covers withdrawal requests. Uncovered withdrawals will be satisfied via validator exits through VEBO.

![Deposits reserve example 2](./assets/lip-35/deposits_reserve_example_2.png)

**Example 3:** An ETH amount exceeding both the deposits reserve target and total withdrawal requests is available in buffer. In this case, the deposits reserve is filled, all withdrawal requests are covered, and the remaining ETH is available for additional CL deposits.

![Deposits reserve example 3](./assets/lip-35/deposits_reserve_example_3.png)

### Edge Case — Last Withdrawals
The deposits reserve is a soft allocation. The Accounting Oracle daemon normally uses the withdrawals reserve as its finalization budget, and the withdrawals reserve is reduced by the deposits reserve (see the allocation formula above). When nearly all stETH is being withdrawn and the buffer must cover the remaining obligations, this default budget may be too small to finalize all outstanding requests.

If TVL drops significantly and the protocol enters a shutting-down state, the DAO should set the deposits reserve target to zero. However, during a Dual Governance rage quit, governance is frozen and cannot adjust the reserves.

**Off-chain fallback.** The deposits reserve does not constrain withdrawal finalization. In this edge case the daemon can raise its finalization budget from `lido.getWithdrawalsReserve()` to the full `lido.getBufferedEther()`. For this to be safe, deposits to the Beacon Chain must also be blocked — the DSM and the depositor bot (deposits and top-ups) are turned off or staking is paused, or there are no depositable validators — otherwise the buffer could be consumed by deposits before finalization.

## Module Shares Easy Track

It is proposed that an [Easy Track (ET)](https://github.com/lidofinance/easy-track) factory be developed to create ET motions to change `stakeShareLimit` and `priorityExitShare` parameters for Lido staking modules. This factory streamlines operations around staking modules' share limits. The factory operates within pre-defined limits set upon deployment, including the ID of the module it works with (one factory per module). `stakeShareLimit` and `priorityExitShare` can be increased or decreased using the same factory.

### Limits

Upon deployment, the following limits are set:

- `stakingModuleId` - the ID of the module the factory will work with
- `maxStakeShareLimitIncrease` - allowed absolute increase of the module's `stakeShareLimit` in BP per 1 ET motion
- `maxStakeShareLimitDecrease` - allowed absolute decrease of the module's `stakeShareLimit` in BP per 1 ET motion
- `maxPriorityExitShareIncrease` - allowed absolute increase of the module's `priorityExitShare` in BP per 1 ET motion
- `maxPriorityExitShareDecrease` - allowed absolute decrease of the module's `priorityExitShare` in BP per 1 ET motion

### Algorithm

![Module Shares Easy Track algorithm](./assets/lip-35/module_shares_easytrack_algorithm.png)

1. **Motion is started**
   An ET motion is created with the desired `stakeShareLimit` and `priorityExitShare` values.
2. **Current state is validated**
   The current module parameters are fetched from `StakingRouter`, and it is verified that:
   - Provided `currentXXX` values match the on-chain data
   - Requested changes are within configured limits
3. **Motion enactment is requested**
   The created motion is submitted for enactment via Easy Track.
4. **State is re-checked and changes are applied**
   The current parameters are fetched again to ensure they have not changed during the motion.
   If unchanged, the new values are applied; otherwise, the transaction is reverted.

### Motion creation

The factory accepts the following parameters for motion creation:

```
    uint256 currentStakeShareLimit,
    uint256 newStakeShareLimit,
    uint256 currentPriorityExitShareThreshold,
    uint256 newPriorityExitShareThreshold
```

These parameters are validated against the current parameters of the staking module to ensure that changes to both parameters are within the [limits](#limits) set upon ET factory deployment and that the current values are provided correctly.

The `currentXXX` values are required to ensure that the motion can be enacted only if the parameters have not been changed while the motion is in progress. This guarantees that concurrent motions cannot be executed simultaneously and that effectively only one motion can be in progress at any moment.

Once all of the actions above are done, the new motion is created.

#### Motion enacting

After enacting the motion, the parameters are rechecked to ensure the `currentXXX` values have not changed. If the checks are successful, the motion is enacted.

### Required changes to StakingRouter

In the current version of the [`StakingRouter.sol`](https://github.com/lidofinance/core/blob/f7916decdddef32c404d47e8e589ee31cc713a56/contracts/0.8.9/StakingRouter.sol) contract module, parameters are changed using a single method, [`updateStakingModule`](https://github.com/lidofinance/core/blob/f7916decdddef32c404d47e8e589ee31cc713a56/contracts/0.8.9/StakingRouter.sol#L296). This approach requires the caller to grant significant permission (`STAKING_MODULE_MANAGE_ROLE`). Given the limited scope of the parameters involved in the described ET factory, it is reasonable to create a distinct method that will allow for changing the following module's parameters:

```solidity
interface IStakingRouter {
  /// @notice Updates staking module share params.
  /// @param _stakingModuleId Staking module id.
  /// @param _stakeShareLimit New stake share limit value.
  /// @param _priorityExitShareThreshold New priority exit share threshold value.
  /// @dev The function is restricted to the `STAKING_MODULE_SHARE_MANAGE_ROLE` role.
  function updateModuleShares(
    uint256 _stakingModuleId,
    uint16 _stakeShareLimit,
    uint16 _priorityExitShareThreshold
  ) external;
}
```

In addition to a distinct method, it is proposed to create a special role (`STAKING_MODULE_SHARE_MANAGE_ROLE`) to allow for granular permissions in the updated version of the `StakingRouter.sol`.

## Proposed params and roles

Below is a list of roles, configuration values, and deployment parameters proposed to be assigned as part of the upcoming upgrade. If certain parameters are not listed, they would either remain unchanged or are defined by network-level constraints.

### Lido

New implementation; state migration is performed via `finalizeUpgrade_v4()` which reads from storage and takes no arguments. The Lido app implementation has no parameters that change between versions.

A new role `BUFFER_RESERVE_MANAGER_ROLE` is created on Lido and granted to the Aragon Agent during the upgrade.

| Role                          | Assignee     |
| ----------------------------- | ------------ |
| `BUFFER_RESERVE_MANAGER_ROLE` | Aragon Agent |

The deposits reserve target governs how much buffered ETH is held back from withdrawal finalization for CL deposits.

| Name                        | Value                                       | Description                                            |
| --------------------------- | ------------------------------------------- | ------------------------------------------------------ |
| `lidoDepositsReserveTarget` | `1000000000000000000000` (1000 ETH in wei) | Deposits reserve target  |

### LidoLocator

New implementation. Two new addresses (`consolidationGateway`, `topUpGateway`) are added to the locator config;

| Name                   | Value                                 | Description                    |
| ---------------------- | ------------------------------------- | ------------------------------ |
| `consolidationGateway` | New `ConsolidationGateway` deployment | non-proxy ConsolidationGateway |
| `topUpGateway`         | New `TopUpGateway` proxy              | TopUpGateway proxy             |

### StakingRouter

Two new immutables (`MAX_EFFECTIVE_BALANCE_WC_TYPE_01`, `MAX_EFFECTIVE_BALANCE_WC_TYPE_02`) are added. Per-module state migration and OpenZeppelin AccessControl role re-import are performed in `finalizeUpgrade_v4(_maxTopUpPerBlockGwei)`, which also seeds the global per-block top-up cap.

| Role                               | Assignee                     |
| ---------------------------------- | ---------------------------- |
| `STAKING_MODULE_SHARE_MANAGE_ROLE` | `EasyTrackEVMScriptExecutor` |

Constructor parameters:

| Name          | Value                                   | Description                                             |
| ------------- | --------------------------------------- | ------------------------------------------------------- |
| `_maxEBType1` | `32000000000000000000` (32 ETH in wei)  | Max effective balance for `0x01` withdrawal credentials |
| `_maxEBType2` | `2048000000000000000000` (2048 ETH wei) | Max effective balance for `0x02` withdrawal credentials |

`finalizeUpgrade_v4(...)` parameters:

| Name                    | Value                      | Description                                              |
| ----------------------- | -------------------------- | -------------------------------------------------------- |
| `_maxTopUpPerBlockGwei` | `3200000000000` (3200 ETH) | Maximum total ETH that can be topped up in a single block |

### AccountingOracle

New implementation. Constructor signature is unchanged; the upgrade calls `finalizeUpgrade_v5(consensusVersion)` to bump the consensus version.

| Name               | Value                                | Description                             |
| ------------------ | ------------------------------------ | --------------------------------------- |
| `consensusVersion` | `6` (passed to `finalizeUpgrade_v5`) | New consensus version bumped on upgrade |

### ValidatorsExitBusOracle

New implementation. The upgrade calls `finalizeUpgrade_v3(maxValidatorsPerReport, maxExitBalanceEth, balancePerFrameEth, frameDurationInSec, consensusVersion)` to bump the consensus version and seed the exit-rate limit state.

| Name                     | Value     | Description                                                      |
| ------------------------ | --------- | ---------------------------------------------------------------- |
| `maxValidatorsPerReport` | `600`     | Maximum number of validator exit requests per oracle report      |
| `maxExitBalanceEth`      | `358400`  | Maximum cumulative ETH of validators referenced in exit requests |
| `balancePerFrameEth`     | `32`      | ETH amount restored to the exit-balance bucket per frame         |
| `frameDurationInSec`     | `48`      | Duration of each refill frame, in seconds                        |
| `consensusVersion`       | `5`       | New consensus version bumped on upgrade                          |

### WithdrawalVault

Constructor is extended with `_consolidationGateway`; `finalizeUpgrade_v3()` bumps the contract version and has no parameters.

| Name                             | Value                                        | Description                                                                         |
| -------------------------------- | -------------------------------------------- | ----------------------------------------------------------------------------------- |
| `_triggerableWithdrawalsGateway` | New TriggerableWithdrawalsGateway deployment | TriggerableWithdrawalsGateway                                                       |
| `_consolidationGateway`          | New `ConsolidationGateway` deployment        | ConsolidationGateway                                                                |
| `_withdrawalRequest`             | `0x00000961Ef480Eb55e80D19ad83579A64c007002` | [EIP-7002](https://eips.ethereum.org/EIPS/eip-7002) withdrawal request predeploy    |
| `_consolidationRequest`          | `0x0000BBdDc7CE488642fb579F8B00f3a590007251` | [EIP-7251](https://eips.ethereum.org/EIPS/eip-7251) consolidation request predeploy |

### DepositSecurityModule

New non-proxy contract deployed at upgrade; its address is written to `LidoLocator`. Ownership is transferred to Aragon Agent; guardians and quorum are re-imported from the previous DSM.

### OracleReportSanityChecker

New non-proxy contract deployed at upgrade; replaces the previous sanity checker in `LidoLocator`.

| Role                 | Assignee     |
| -------------------- | ------------ |
| `DEFAULT_ADMIN_ROLE` | Aragon Agent |

Constructor parameters (new / changed limits passed in the `_limits` struct):

| Name                                  | Value    | Description                                                           |
| ------------------------------------- | -------- | --------------------------------------------------------------------- |
| `maxEffectiveBalanceWeightWCType01`   | `32`     | Per-key weight in ETH for `0x01` keys used in VEBO check              |
| `maxEffectiveBalanceWeightWCType02`   | `2048`   | Per-key weight in ETH for `0x02` keys used in VEBO check              |
| `maxCLBalanceDecreaseBP`              | `360`    | Max CL balance decrease over 36-day window (BP, 3.6%)                 |
| `consolidationEthAmountPerDayLimit`   | `93375`  | Max ETH consolidated per day                                          |
| `exitedValidatorEthAmountLimit`       | `32`     | Per-validator ETH amount used in exit reporting                       |
| `externalPendingBalanceCapEth`        | `300`    | Extra external pending balance cap for bounded side deposits / top-ups |
| `exitedEthAmountPerDayLimit`          | `57600`  | Max ETH amount of validators that may exit per day                    |
| `appearedEthAmountPerDayLimit`        | `57600`  | Max ETH amount of validators that may appear per day                  |
| `maxBalanceExitRequestedPerReportInEth` | `19200` | Max ETH amount of validators referenced in exits within one report   |

### ConsolidationGateway

New non-proxy contract.

| Role                             | Assignee                      |
| -------------------------------- | ----------------------------- |
| `DEFAULT_ADMIN_ROLE`             | Aragon Agent                  |
| `PAUSE_ROLE`                     | CircuitBreaker, ResealManager |
| `RESUME_ROLE`                    | ResealManager                 |
| `ADD_CONSOLIDATION_REQUEST_ROLE` | ConsolidationBus              |
| `EXIT_LIMIT_MANAGER_ROLE`        | Not assigned by default       |

Constructor parameters:

| Name                            | Value                                                                | Description                                                   |
| ------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------- |
| `admin`                         | Aragon Agent (granted via temporary admin after `completeSetup`)     | Receives `DEFAULT_ADMIN_ROLE`                                 |
| `lidoLocator`                   | LidoLocator proxy                                                    | Lido protocol service locator                                 |
| `maxConsolidationRequestsLimit` | `2900`                                                               | Max consolidation requests accepted before rate-limit applies |
| `consolidationsPerFrame`        | `1`                                                                  | Number of consolidations processed per frame                  |
| `frameDurationInSec`            | `30`                                                                 | Frame duration used for limit refill (seconds)                |
| `gIFirstValidatorPrev`          | `0x0000000000000000000000000000000000000000000000000096000000000028` | Generalized index for first validator before fork pivot       |
| `gIFirstValidatorCurr`          | `0x0000000000000000000000000000000000000000000000000096000000000028` | Generalized index for first validator after fork pivot        |
| `pivotSlot`                     | `0`                                                                  | Slot at which the active generalized index switches           |

### ConsolidationGateway CircuitBreaker registration

`ConsolidationGateway` is registered as a pausable contract on the existing [CircuitBreaker](./lip-34.md) instance, with a dedicated pauser committee assigned to it. Pause duration and heartbeat interval are global CircuitBreaker parameters and are not set per registration.

| Name       | Value                    | Description                                                      |
| ---------- | ------------------------ | ---------------------------------------------------------------- |
| `pausable` | `ConsolidationGateway`   | Contract registered as pausable on CircuitBreaker                |
| `pauser`   | Gate Seal contract | Gate Seal committee authorized to trigger a pause on `ConsolidationGateway` |

### ConsolidationBus

New contract behind `OssifiableProxy`.

| Role                 | Assignee              |
| -------------------- | --------------------- |
| Proxy admin          | Aragon Agent          |
| `DEFAULT_ADMIN_ROLE` | Aragon Agent          |
| `PUBLISH_ROLE`       | ConsolidationMigrator |
| `REMOVE_ROLE`        | Aragon Agent          |
| `MANAGE_ROLE`        | Aragon Agent          |

Constructor parameters (implementation):

| Name                   | Value                            | Description                                   |
| ---------------------- | -------------------------------- | --------------------------------------------- |
| `consolidationGateway` | New ConsolidationGateway address | Downstream gateway receiving executed batches |

`initialize(...)` parameters (called on proxy):

| Name                      | Value                              | Description                                                 |
| ------------------------- | ---------------------------------- | ----------------------------------------------------------- |
| `admin`                   | Aragon Agent (via temporary admin) | Receives `DEFAULT_ADMIN_ROLE`, `MANAGE_ROLE`, `REMOVE_ROLE` |
| `initialBatchSize`        | `350`                              | Maximum number of consolidation requests per batch          |
| `initialMaxGroupsInBatch` | `10`                               | Maximum number of target validator groups per batch         |
| `initialExecutionDelay`   | `86400` (24 hours)                 | Delay (seconds) between batch publishing and executability  |

### ConsolidationMigrator

New contract behind `OssifiableProxy`. Source/target module IDs are immutable constructor args.

| Role                 | Assignee                    |
| -------------------- | --------------------------- |
| Proxy admin          | Aragon Agent                |
| `DEFAULT_ADMIN_ROLE` | Aragon Agent                |
| `ALLOW_PAIR_ROLE`    | EasyTrack EVMScriptExecutor |
| `DISALLOW_PAIR_ROLE` | CMC Committee               |

Constructor parameters (implementation):

| Name               | Value                  | Description                                           |
| ------------------ | ---------------------- | ----------------------------------------------------- |
| `stakingRouter`    | StakingRouter proxy    | Used to resolve module by id and introspect operators |
| `consolidationBus` | ConsolidationBus proxy | Bus that receives published consolidation batches     |
| `_sourceModuleId`  | `1` (NOR / CMv1)       | Immutable source module ID                            |
| `_targetModuleId`  | `4` (CMv2)             | Immutable target module ID                            |

`initialize(...)` parameters (called on proxy):

| Name    | Value                              | Description                   |
| ------- | ---------------------------------- | ----------------------------- |
| `admin` | Aragon Agent (via temporary admin) | Receives `DEFAULT_ADMIN_ROLE` |

### TopUpGateway

New contract behind `OssifiableProxy`.

| Role                 | Assignee                      |
| -------------------- | ----------------------------- |
| Proxy admin          | Aragon Agent                  |
| `DEFAULT_ADMIN_ROLE` | Aragon Agent                  |
| `PAUSE_ROLE`         | CircuitBreaker, ResealManager |
| `RESUME_ROLE`        | ResealManager                 |
| `TOP_UP_ROLE`        | Lido Depositor Bot            |
| `MANAGE_LIMITS_ROLE` | Not assigned by default       |

Constructor parameters (implementation):

| Name                    | Value                                                                | Description                                             |
| ----------------------- | -------------------------------------------------------------------- | ------------------------------------------------------- |
| `_lidoLocator`          | LidoLocator proxy                                                    | Lido protocol service locator                           |
| `_gIFirstValidatorPrev` | `0x0000000000000000000000000000000000000000000000000096000000000028` | Generalized index for first validator before fork pivot |
| `_gIFirstValidatorCurr` | `0x0000000000000000000000000000000000000000000000000096000000000028` | Generalized index for first validator after fork pivot  |
| `_pivotSlot`            | `0`                                                                  | Slot at which the active generalized index switches     |
| `_slotsPerEpoch`        | `32`                                                                 | Slots per epoch (from chain spec)                       |

`initialize(...)` parameters (called on proxy):

| Name                     | Value                              | Description                                                                               |
| ------------------------ | ---------------------------------- | ----------------------------------------------------------------------------------------- |
| `_admin`                 | Aragon Agent (via temporary admin) | Receives `DEFAULT_ADMIN_ROLE`                                                             |
| `_maxValidatorsPerTopUp` | `32`                               | Maximum number of validators a single `topUp` can process                                 |
| `_minTopUpBlockDistance`      | `75`                                | Minimum block distance between `topUp` calls                                              |
| `_maxRootAgeSec`         | `600`                              | Maximum age (seconds) of the beacon root used to prove validator state                    |
| `_targetBalanceGwei`     | `2046750000000` (2046.75 ETH)      | Validator target balance ceiling after top-up (leaves 1.25 ETH safety margin below MaxEB) |
| `_minTopUpGwei`          | `2000000000` (2 ETH)               | Minimum top-up amount; smaller calculated top-ups are skipped                             |

### TopUpGateway CircuitBreaker registration

`TopUpGateway` is registered as a pausable contract on the existing [CircuitBreaker](./lip-34.md) instance, with a dedicated pauser committee assigned to it. Pause duration and heartbeat interval are global CircuitBreaker parameters and are not set per registration.

| Name       | Value              | Description                                                            |
| ---------- | ------------------ | ---------------------------------------------------------------------- |
| `pausable` | `TopUpGateway`     | Contract registered as pausable on CircuitBreaker                      |
| `pauser`   | Gate Seal contract | Gate Seal committee authorized to trigger a pause on `TopUpGateway`    |

### TriggerableWithdrawalGateway

Parameter upgrade for `TriggerableWithdrawalGateway`:

| Name                   | Value | Description                                                     |
| ---------------------- | ----- | --------------------------------------------------------------- |
| `maxExitRequestsLimit` | 250   | Maximum number of triggerable exit requests in the limit bucket |
| `exitsPerFrame`        | 1     | Number of exit requests restored to the bucket per frame        |
| `frameDurationInSec`   | 240   | Duration of each refill frame, in seconds                       |

The upgrade also grants `TW_EXIT_LIMIT_MANAGER_ROLE` on `TriggerableWithdrawalsGateway` to the Aragon Agent (so the Agent can call `setExitRequestLimit` during the upgrade and afterward).

| Role                         | Assignee     |
| ---------------------------- | ------------ |
| `TW_EXIT_LIMIT_MANAGER_ROLE` | Aragon Agent |

### EasyTrack

Two new factories are proposed to be registered; EasyTrack admin roles would remain unchanged.

| Factory                          | Permission target                  |
| -------------------------------- | ---------------------------------- |
| `UpdateStakingModuleShareLimits` | `StakingRouter.updateModuleShares` |
| `AllowConsolidationPair`         | `ConsolidationMigrator.allowPair`  |

## Security Considerations

### On-chain proofs as the trust anchor for top-ups and consolidations

Both new flows that move ETH between the protocol and validators — **top-ups** via `TopUpGateway` and **consolidations** via `ConsolidationGateway` — anchor their security on Merkle proofs of consensus-layer state verified against [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788) beacon roots, rather than on trust in the calling actor.

For consolidations, the `ConsolidationGateway` verifies the **target** validator's withdrawal credentials via on-chain CL proofs (see [Consolidation Gateway](#consolidation-gateway)), so consolidation cannot redirect stake to a validator outside the Lido WC set. The **source** withdrawal credentials are guaranteed by the EIP-7251 system contract itself, since the consolidation request is signed by the `WithdrawalVault` contract, which is the canonical withdrawal address for all Lido core validators.

For top-ups, the `TopUpGateway` verifies the target validator's withdrawal credentials via on-chain CL proofs (see [Validator State Validation](#validator-state-validation)), so a top-up cannot be routed to a non-Lido validator — even in the event a front-run during the predeposit was not caught by the DSM. The per-validator top-up limit is additionally forced to zero for slashed, exiting, or already-exited validators.

### Authorized Depositor Bot for Top-Ups

The `topUp` method is gated by the `TOP_UP_ROLE`, which is granted exclusively to the Lido Depositor Bot. This permissioning complements on-chain proof verification: the bot provides inputs that are not feasible to verify on-chain and applies validator selection logic that does not affect protocol safety.

**Risks if the bot malfunctions or is compromised:**

- **Over-deposits due to unreported pending deposits.** [Pending deposits](#validator-pending-balance) are not verified on-chain—the bot supplies them off-chain. If queued deposits are omitted, a validator may temporarily exceed its `MaxEB`.
- **Top-ups to soon-to-be consolidated validators.** The bot should not top up validators that are referenced as targets in `pending_consolidations`, as this may cause their balance to temporarily exceed `MaxEB`.
- **Imbalance between validator cohorts.** The bot should maintain a sufficient pool of top-up–eligible validators so that buffered ETH can be utilized without waiting for new validator activations. Over-prioritizing top-ups over initial deposits can deplete this pool and stall buffer utilization.
- **Suboptimal top-up distribution.** Evenly distributing top-ups across partially funded validators may reduce APR.

**Mitigations**

Excessive or anomalous top-ups are detected by existing monitoring and trigger an incident response, during which the authorized actor can be paused. Combined with the ability to pause the TopUpGateway, the per-block top-up cap (`maxTopUpPerBlockGwei`), the `minTopUpBlockDistance` constraint, and automatic skimming (excess is automatically skimmed to the validator’s WC), these measures bound the worst-case impact of bot misbehavior to a recoverable inefficiency rather than a loss of protocol funds.

### Deposit reserve and withdrawal demand

The deposit reserve protects a portion of buffered ether for CL deposits, reserve sizing is a governance trade-off:

- A reserve that is too **small** does not meaningfully help with stake rebalancing during periods of high withdrawal demand.
- A reserve that is too **large** delays withdrawal request finalization, since reserved ETH is filled with the highest priority and is not available to satisfy unfinalized stETH (see [Buffered Ether Allocation](#buffered-ether-allocation)).

The reserve target is configurable by `BUFFER_RESERVE_MANAGER_ROLE` so it can be tuned over time.

## References

- [LIP-33. Community Staking Module v3 and Curated Module v2](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-33.md)
- [Hoodi Curated Module v2 Migration - Node Operator Overview](https://enchanted-direction-844.notion.site/Hoodi-Curated-Module-v2-Migration-Node-Operator-Overview-PUBLIC-880bf633d0c982ea9dc58158d876d9e3?source=copy_link)
- [ConsolidationGateway Limits](https://docs.google.com/document/d/1sq-CVq0AAznt0I7uamX0IWHzzaX3iGc3xfghDtB-9DA/edit?tab=t.0)
- [TriggerableWithdrawalsGateway Limits](https://hackmd.io/@5wamg-wlRCCzGh0aoCqR0w/SJ-bhZ5elx/edit)
- [Sanity checks specification](https://docs.google.com/document/d/1YAWLxZk90dkcwCeQkv8xSeWYfVNKGCJmicZpPn-aGR0/edit?tab=t.0)
- [CL balance decrease calculations](https://docs.google.com/document/d/1MK9XMU-xVdw0XQG9cxtR0rxusuBr1CI4DuGNxNXgswI/edit?tab=t.0)
- [DG Reseal Manager](https://github.com/lidofinance/dual-governance/blob/main/contracts/ResealManager.sol)
