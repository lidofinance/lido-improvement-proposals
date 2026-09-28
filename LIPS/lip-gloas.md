---
lip: <to be assigned>
title: Ensuring Compatibility with Ethereum’s Glamsterdam Upgrade
status: WIP
author: TBA
discussions-to: <create a thread on https://research.lido.fi/ and drop the link here>
created: 2026-07-14
updated: 2026-07-14
---

# Ensuring Compatibility with Ethereum’s Glamsterdam Upgrade

## Table of Contents

- [Simple Summary](#simple-summary)
- [Abstract](#abstract)
- [Motivation](#motivation)
- [Specification](#specification)
  - [1. Report Read Slot](#1-report-read-slot)
    - [Overview](#overview)
    - [Rationale](#rationale)
      - [In-Flight Withdrawal Add-Back](#in-flight-withdrawal-add-back)
    - [Technical Specification](#technical-specification)
  - [2. Accounting Oracle](#2-accounting-oracle)
    - [Overview](#overview-1)
    - [Rationale](#rationale-1)
      - [Bunker Mode Sample Identity](#bunker-mode-sample-identity)
      - [Staking Vault Report Timestamp](#staking-vault-report-timestamp)
    - [Technical Specification](#technical-specification-1)
  - [3. Validator Exit Bus Oracle](#3-validator-exit-bus-oracle)
    - [Overview](#overview-2)
    - [Rationale](#rationale-2)
      - [Exit Churn Limit](#exit-churn-limit)
      - [Sweep Cycle Prediction](#sweep-cycle-prediction)
      - [In-Flight Withdrawals in Predicted Liquidity](#in-flight-withdrawals-in-predicted-liquidity)
    - [Technical Specification](#technical-specification-2)
  - [4. Beacon API Compatibility](#4-beacon-api-compatibility)
    - [Overview](#overview-3)
    - [Rationale](#rationale-3)
    - [Technical Specification](#technical-specification-3)
  - [5. Oracle Consensus Versions](#5-oracle-consensus-versions)
    - [Overview](#overview-4)
    - [Rationale](#rationale-4)
    - [Technical Specification](#technical-specification-4)
- [Security Considerations](#security-considerations)
- [Failure Modes](#failure-modes)
- [Links](#links)
- [Copyright](#copyright)

## Simple Summary

This proposal outlines changes to the Lido protocol to ensure continuous operation with Ethereum’s upcoming Glamsterdam hardfork, focusing exclusively on compatibility.

## Abstract

For the off-chain components of the [Accounting](https://docs.lido.fi/guides/oracle-spec/accounting-oracle), [Validator Exit Bus](https://docs.lido.fi/guides/oracle-spec/validator-exit-bus), and performance oracles of the staking modules, this proposal:

- Builds every report from the first non-missed child state of `ref_slot`, while retaining `ref_slot` as the report's time reference;
- Adds withdrawals deducted on the consensus layer but not yet credited on the execution layer back to Lido CL balances;
- Identifies bunker-mode balance samples by CL state root instead of EL block number;
- Switches the Validator Exit Bus Oracle to the uncapped exit churn limit, with parameters read from the network config;
- Remodels withdrawal capacity in the sweep-cycle prediction;
- Updates Beacon API handling and oracle consensus versions.

## Motivation

[EIP-7732 (ePBS)](https://eips.ethereum.org/EIPS/eip-7732) separates the execution payload from the beacon block: a slot's payload may be revealed late or withheld. The oracles currently assume the reference slot's own EL block is always available. After the hardfork.

[EIP-8061](https://eips.ethereum.org/EIPS/eip-8061) removes the cap on the exit churn limit. Keeping the old formula overestimates the exit queue wait about five times and makes the Validator Exit Bus Oracle request fewer exits than needed.

Two Beacon API changes also affect the oracles: the proposer-duties `dependent_root` semantics introduced by [EIP-7917](https://eips.ethereum.org/EIPS/eip-7917), and the rename of `SECONDS_PER_SLOT` to `SLOT_DURATION_MS` in the node config.

## Specification

### 1. Report Read Slot

#### Overview

For a report with a post-Gloas `ref_slot`, all three oracle families build their reports from one beacon state: the first non-missed child of `ref_slot`. The EL block for the report is the `state.latest_block_hash` of that same state. As a required step of this read, every oracle adds in-flight withdrawals of Lido validators back to their CL balances.

#### Rationale

Under Gloas, the latest confirmed EL block and the deposits from a slot's payload appear in beacon state only when a child block is processed. Reading the first non-missed child gives a consistent snapshot of balances, pending deposits, and the matching EL block from one state.

This change has three consequences:

- **Finality wait.** The oracle must wait for the child state to finalize, which usually takes about one epoch (~6.4 minutes) longer than waiting for `ref_slot`.
- **One-time rebase shift.** The first report after the switch covers one additional epoch. For a daily report, its rebase is therefore about `1/225 ≈ 0.44%` higher. This positive rebase is within the sanity-checker limits; later reports return to their normal span.
- **Time reference.** All derived timestamps (event lookback windows and staking vault report metadata) must stay tied to `ref_slot`. If they used the read slot instead, the lookback window would widen by at least one slot and skewing reward-rate averages.

##### In-Flight Withdrawal Add-Back

In the child state, validator balances already have the expected withdrawals deducted, but the EL block has not credited them to the Withdrawal Vault yet. The specification exposes this amount as `state.payload_expected_withdrawals`.

#### Technical Specification

- The algorithm is selected from the fork version of `ref_slot`. 
  - A report with a pre-Gloas `ref_slot` uses the legacy path even when it is built or submitted after activation.
  - For a report with a post-Gloas `ref_slot`, the read slot is the first non-missed child of `ref_slot`.
- The oracle waits for the read slot to be finalized before building a report.
- All CL data is read from the read slot's state. All EL data is read at the block referenced by that state's `latest_block_hash`.
- `ref_slot` stays in the report and remains the basis for all timestamps. The event lookback cutoff is computed from `ref_slot` and the genesis time instead of the anchor block timestamp.
- The `payload_expected_withdrawals` amounts of Lido validators, summed per validator index, are added back to the CL balances read from the read slot's state. This step is required for all oracles.

### 2. Accounting Oracle

#### Overview

It is proposed to:

- Identify bunker-mode samples by CL state root;
- Derive the staking vault report timestamp from `ref_slot`.

#### Rationale

The in-flight withdrawal add-back from [§1](#in-flight-withdrawal-add-back) is applied everywhere Lido CL balances are summed: total CL balance, the per-module breakdown, bunker-mode rebase (both samples), and staking vault Total Value. Without per-index summing, a vault's Total Value, Merkle leaf, and fee come out too low while global totals still look correct.

##### Bunker Mode Sample Identity

With withheld payloads, several finalized CL states can share the same EL block while CL rewards and penalties keep accruing. The current rebase check returns zero whenever two samples have the same EL block number, which hides a negative rebase during exactly the disruption bunker mode is meant to detect. Samples are therefore compared by state root. The vault withdrawal lookup returns zero when there are no EL blocks between the samples.

##### Staking Vault Report Timestamp

The staking vault IPFS report takes its `timestamp` from the read slot, while `AccountingOracle` uses `refSlot`. After the hardfork, the two would differ on every report, so the timestamp is derived from `ref_slot`.

#### Technical Specification

- The in-flight withdrawal add-back from §1 is applied to the total and per-module CL balances.
- For staking vaults, the amounts are summed per validator index and added to that validator's balance.
- In bunker mode, each balance sample is corrected from its own state, and samples are compared by state root. Vault withdrawals between two samples are zero when there are no EL blocks between them.
- The staking vault IPFS report timestamp is computed from `ref_slot`.

### 3. Validator Exit Bus Oracle

#### Overview

This proposal:

- Switch to the uncapped exit churn limit after the hardfork;
- Exclude pending partial withdrawals from the sweep cycle prediction.

#### Rationale

##### Exit Churn Limit

EIP-8061 replaces the capped exit churn limit (256 ETH/epoch) with an uncapped one, which gives about 1220 ETH/epoch at ~40M ETH staked. With the old formula, the oracle overestimates withdrawal epochs and predicted liquidity, so it requests fewer exits than needed. `CHURN_LIMIT_QUOTIENT_GLOAS` and `MIN_PER_EPOCH_CHURN_LIMIT_ELECTRA` differ across networks, so they are read from the node config instead of being hard-coded. If a parameter is missing after the hardfork, the oracle fails loudly instead of using a default.

##### Sweep Cycle Prediction

After the hardfork, pending partial withdrawals take priority over the validator sweep. That queue can be filled by anyone, and a snapshot of it does not predict future load. Excluding it from the prediction errs on the safe side: the oracle may request slightly more exits.

Builder pending withdrawals and the builder sweep also consume the per-payload withdrawal capacity before the validator sweep. The prediction must include their current queues and position in that shared capacity. An empty parent payload makes no withdrawal-pointer progress. Because future builder load and empty-parent periods have no protocol-level upper bound, the implementation must use and expose a documented operational assumption or cap for them.

#### Technical Specification

- For exits processed under Gloas, the per-epoch exit churn is `max(MIN_PER_EPOCH_CHURN_LIMIT_ELECTRA, total_active_balance // CHURN_LIMIT_QUOTIENT_GLOAS)`, rounded down to `EFFECTIVE_BALANCE_INCREMENT`, with no upper cap. For pre-Gloas processing, the current formula is used.
- Both parameters are read from the node config. If either is missing when the Gloas rule applies, the Validator Exit Bus Oracle stops instead of using a default.
- At the fork boundary, the selected churn rule must match the consensus rules when the voluntary-exit message is first processed. The report's `ref_slot` alone must not select the rule for an exit that may be processed after activation. The rollout must either prevent such a pre-fork request from being first processed after activation, or model it with the Gloas rule.
- The sweep cycle prediction ignores pending partial withdrawals after the hardfork.
- The sweep cycle prediction includes the current builder queues and treats an observed empty parent payload as zero withdrawal-pointer progress. It uses a documented operational assumption or cap for future builder load and empty-parent periods.
- Predicted EL liquidity includes in-flight withdrawals of Lido validators once as a pending EL inflow. The corresponding CL validator balances are not restored.

```python
def get_exit_churn_limit(total_active_balance, min_per_epoch_churn_limit, churn_limit_quotient) -> Gwei:
    churn = max(min_per_epoch_churn_limit, total_active_balance // churn_limit_quotient)
    return Gwei(churn - churn % EFFECTIVE_BALANCE_INCREMENT)

if is_gloas:
    min_churn, quotient = self.w3.cc.get_config_spec().gloas_exit_churn_params()  # Raises if missing.
    per_epoch_churn = get_exit_churn_limit(total_active_balance, min_churn, quotient)
else:
    per_epoch_churn = get_activation_exit_churn_limit(total_active_balance)
```

### 4. Beacon API Compatibility

#### Overview

This proposal supports the v2 proposer-duties endpoint at Gloas and the `SLOT_DURATION_MS` config field. The config-field change is independent of the Gloas activation date.

#### Rationale

- **Proposer duties.** EIP-7917 introduced deterministic proposer lookahead. The Beacon API v2 proposer-duties endpoint defines the `dependent_root` needed for Gloas; the v1 endpoint is deprecated and implementations do not uniformly provide the required semantics. The oracle treats a `dependent_root` mismatch as fatal, so this affects the performance collector shared by the CSM, CSM `0x02`, and CM v2 oracles. Some clients may not support v2 when the release is deployed.
- **Slot duration.** [consensus-specs#4926](https://github.com/ethereum/consensus-specs/pull/4926) renames `SECONDS_PER_SLOT` to `SLOT_DURATION_MS`. The oracle will fail to parse the config of any node that drops the old field.

#### Technical Specification

- For post-Gloas duties, use `GET /eth/v2/validator/duties/proposer/{epoch}`. Its `dependent_root` is `get_block_root_at_slot(state, compute_start_slot_at_epoch(epoch - 1) - 1)` (or the genesis block root on underflow). If a node does not support v2, fall back to v1 and treat a `dependent_root` mismatch as non-fatal. Do not require v2 before Gloas activation.
- Accept either `SLOT_DURATION_MS` or `SECONDS_PER_SLOT`. Raise an error when the two are inconsistent or when the duration is not a whole number of seconds.
- Read the churn parameters for §3 from the same node config.

## Links

- [Gloas consensus specification](https://github.com/ethereum/consensus-specs/tree/master/specs/gloas)
- EIPs
  - [EIP-7732: Enshrined Proposer-Builder Separation](https://eips.ethereum.org/EIPS/eip-7732)
  - [EIP-8061: Exit churn](https://eips.ethereum.org/EIPS/eip-8061)
  - [EIP-7917: Deterministic proposer lookahead](https://eips.ethereum.org/EIPS/eip-7917)
- [beacon-APIs#563: proposer duties v2](https://github.com/ethereum/beacon-APIs/pull/563)
- [consensus-specs#4926: `SLOT_DURATION_MS`](https://github.com/ethereum/consensus-specs/pull/4926)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

<!-- my hat
       ⧊
      / \
 ____/   \____
/_____________\
 |   |    |   |
 ◌   ◌    ◌   ◌
-->

