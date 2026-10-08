---
lip: 43
title: Ensuring Compatibility with Ethereum’s Glamsterdam Upgrade
status: draft
author: Dmitry Gusakov (@dgusakov), Dmitry Chernukhin (@madlabman), Raman Siamionau (@F4ever), Alexander H (@hweawer)
discussions-to: <create a thread on https://research.lido.fi/ and drop the link here>
created: 2026-10-01
updated: 2026-10-08
---

# LIP-43: Ensuring Compatibility with Ethereum’s Glamsterdam Upgrade

## Simple Summary

This proposal outlines changes to the Lido protocol to ensure continuous operation with Ethereum’s upcoming [Glamsterdam hardfork](https://eips.ethereum.org/EIPS/eip-7773), focusing exclusively on compatibility.

For the off-chain components of the [Accounting](https://docs.lido.fi/guides/oracle-spec/accounting-oracle), [Validator Exit Bus](https://docs.lido.fi/guides/oracle-spec/validator-exit-bus), and performance oracles of the staking modules, this proposal:

- Builds every report from the first non-missed child state of `ref_slot`, while retaining `ref_slot` as the report's time reference;
- Adds withdrawals deducted on the consensus layer but not yet credited on the execution layer back to Lido CL balances;
- Identifies bunker-mode balance samples by CL state root instead of EL block number;
- Switches the Validator Exit Bus Oracle to the uncapped exit churn limit, with parameters read from the network config;
- Limits the sweep-cycle prediction to validator withdrawals;
- Moves the per-report limit on validator exit requests (lowered from 600 to 500 after gas limit issues) from a hard-coded oracle constant to the `OracleDaemonConfig` contract;
- Updates Beacon API handling and oracle consensus versions.

On-chain changes comprise only changes to `Verifier` contracts and contract components, namely:
- Verifiers now use hard-coded generalized indices (GIndices), with item paths derived at runtime.
- Proof methods for the events that should be proven against the exact beacon block now use `block_roots` from the corresponding beacon state to reach required block roots.

Also, several protocol parameters used in sanity checks are updated to conform with EIP-8061.

## Motivation

[EIP-7732 (ePBS)](https://eips.ethereum.org/EIPS/eip-7732) separates the execution payload from the beacon block: a slot's payload may be revealed late or withheld. The oracles currently assume the reference slot's own EL block is always available. After the hardfork <TBD>. Verifier contracts use `block_roots` from the corresponding beacon state to reach required block roots for the proofs requiring the exact beacon block.

[EIP-8061](https://eips.ethereum.org/EIPS/eip-8061) removes the cap on the exit churn limit. Keeping the old formula overestimates the exit queue wait about five times and makes the Validator Exit Bus Oracle request fewer exits than needed.

[EIP-7688](https://eips.ethereum.org/EIPS/eip-7688) stabilizes field and item paths across compatible future extensions, allowing verifiers to use hard-coded generalized indices (GIndices) with item paths derived at runtime.

Two Beacon API changes also affect the oracles: the proposer-duties `dependent_root` semantics introduced by [EIP-7917](https://eips.ethereum.org/EIPS/eip-7917), and the rename of `SECONDS_PER_SLOT` to `SLOT_DURATION_MS` in the node config.

## Specification

### 1. Report Read Slot in Oracles

For a report with a post-Gloas `ref_slot`, all three oracle families build their reports from one beacon state: the first non-missed child of `ref_slot`. The EL block for the report is the `state.latest_block_hash` of that same state. As a required step of this read, every oracle adds in-flight withdrawals of Lido validators back to their CL balances.

#### Rationale

Under Gloas, the latest confirmed EL block and the deposits from a slot's payload appear in beacon state only when a child block is processed. Reading the first non-missed child gives a consistent snapshot of balances, pending deposits, and the matching EL block from one state.

This change has three consequences:

- **Finality wait.** The oracle must wait for the child state to finalize, which usually takes about one epoch (~6.4 minutes) longer than waiting for `ref_slot` to get finalized.
- **One-time rebase shift.** The first report after the switch covers one additional epoch. For a daily report, its rebase is therefore about `1/225 ≈ 0.44%` higher. This positive rebase is within the sanity-checker limits; later reports return to their normal span.
- **Time reference.** All derived timestamps (event lookback windows and staking vault report metadata) must stay tied to `ref_slot`. If they used the read slot instead, the lookback window would widen by at least one slot and skewing reward-rate averages.

##### In-Flight Withdrawal Add-Back

In the child state, validator balances already have the expected withdrawals deducted, but the EL block has not credited them to the Withdrawal Vault yet. The specification exposes this amount as `state.payload_expected_withdrawals` so these amounts can be added back to the CL balances for accurate reporting.

#### Technical Specification

- The algorithm is selected from the fork version of `ref_slot`. 
  - A report with a pre-Gloas `ref_slot` uses the legacy path even when it is built or submitted after activation.
  - For a report with a post-Gloas `ref_slot`, the read slot is the first non-missed child of `ref_slot`.
- The oracle waits for the read slot to be finalized before building a report.
- All CL data is read from the read slot's state. All EL data is read at the block referenced by that state's `latest_block_hash`.
- `ref_slot` stays in the report and remains the basis for all timestamps. The event lookback cutoff is computed from `ref_slot` and the genesis time instead of the anchor block timestamp.
- The `payload_expected_withdrawals` amounts of Lido validators, summed per validator index, are added back to the CL balances read from the read slot's state. This step is required for all oracles.

### 2. Accounting Oracle

It is proposed to:

- Identify bunker-mode samples by CL state root;

#### Rationale

The in-flight withdrawal add-back from [§1](#in-flight-withdrawal-add-back) is applied everywhere Lido CL balances are summed: total CL balance, the per-module breakdown, bunker-mode rebase (both samples), and staking vault Total Value. Without per-index summing, a vault's Total Value, Merkle leaf, and fee come out too low while global totals still look correct.

##### Bunker Mode Sample Identity

With withheld payloads, several finalized CL states can share the same EL block while CL rewards and penalties keep accruing. The current rebase check returns zero whenever two samples have the same EL block number, which hides a negative rebase during exactly the disruption bunker mode is meant to detect. Samples are therefore compared by state root. The vault withdrawal lookup returns zero when there are no EL blocks between the samples.

#### Technical Specification

- The in-flight withdrawal add-back from §1 is applied to the total and per-module CL balances.
- In bunker mode, each balance sample is corrected from its own state, and samples are compared by state root. Vault withdrawals between two samples are zero when there are no EL blocks between them.

### 3. Validator Exit Bus Oracle

It is proposed to:

- Switch to the uncapped exit churn limit after the hardfork;
- Count only validator withdrawals in the sweep cycle prediction;
- Read the maximum number of validator exit requests per report from the `OracleDaemonConfig` contract instead of a hard-coded constant.

#### Rationale

##### Exit Churn Limit

EIP-8061 replaces the capped exit churn limit (256 ETH/epoch) with an uncapped one, which gives about 1311 ETH/epoch at ~43M ETH staked. With the old formula, the oracle overestimates withdrawal epochs and predicted liquidity, so it requests fewer exits than needed. `CHURN_LIMIT_QUOTIENT_GLOAS` and `MIN_PER_EPOCH_CHURN_LIMIT_ELECTRA` differ across networks, so they are read from the node config instead of being hard-coded. If a parameter is missing after the hardfork, the oracle fails loudly instead of using a default.

##### Sweep Cycle Prediction

The validator sweep is no the only consumer of the per-payload withdrawal capacity. Builder pending withdrawals, the builder sweep, and pending partial withdrawals are processed before it.

Of these consumers, only the validator sweep is predictable: it walks the validator set in a fixed order at a known rate. The others are periodical. Their load depends on external actors and comes in bursts, so a snapshot of the current queues does not predict future load, and there is no protocol-level upper bound to model them against.

The Validator Exit Bus Oracle does not try to emulate the consensus layer; it only needs a reasonable estimate of when Lido validators will be swept. Therefore, after the hardfork, the sweep cycle prediction counts only validator withdrawals and ignores all other consumers. As a result, VEBO will request to exit slightly more validators in some rare cases.

##### Max Exit Requests per Report

This proposal also closes technical debt from a hotfix. Gas limit issues were found when a report requests the maximum of 600 validator exits, so the limit was lowered to 500 by a constant hard-coded in the oracle. The limit is moved to the `OracleDaemonConfig` contract as `MAX_VALIDATOR_EXIT_REQUESTS_PER_REPORT` with a value of `500`, so it can be adjusted by the DAO without an oracle release.

#### Technical Specification

- For exits processed under Gloas, the per-epoch exit churn is `max(MIN_PER_EPOCH_CHURN_LIMIT_ELECTRA, total_active_balance // CHURN_LIMIT_QUOTIENT_GLOAS)`, rounded down to `EFFECTIVE_BALANCE_INCREMENT`, with no upper cap. For pre-Gloas processing, the current formula is used.
- Both parameters are read from the node config. If either is missing when the Gloas rule applies, the Validator Exit Bus Oracle stops instead of using a default.
- At the fork boundary, the selected churn rule must match the consensus rules when the voluntary-exit message is first processed. The report's `ref_slot` alone must not select the rule for an exit that may be processed after activation. The rollout must either prevent such a pre-fork request from being first processed after activation, or model it with the Gloas rule.
- After the hardfork, the sweep cycle prediction counts only validator withdrawals. Pending partial withdrawals, builder pending withdrawals, the builder sweep, and empty parent payloads are not modeled.
- Predicted EL liquidity includes in-flight withdrawals of Lido validators once as a pending EL inflow. The corresponding CL validator balances are not restored.
- A new `MAX_VALIDATOR_EXIT_REQUESTS_PER_REPORT` key is added to the `OracleDaemonConfig` contract with a value of `500`.
- The Validator Exit Bus Oracle reads this value from `OracleDaemonConfig` and uses it as the maximum number of validators requested for exit in a single report. The hard-coded hotfix constant is removed. If the key is missing, the oracle stops instead of using a default.

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

### 4. Beacon API Compatibility in Oracles

This proposal supports the v2 proposer-duties endpoint at Gloas and the `SLOT_DURATION_MS` config field. The config-field change is independent of the Gloas activation date.

#### Rationale

- **Proposer duties.** EIP-7917 introduced deterministic proposer lookahead. The Beacon API v2 proposer-duties endpoint defines the `dependent_root` needed for Gloas; the v1 endpoint is deprecated and implementations do not uniformly provide the required semantics. The oracle treats a `dependent_root` mismatch as fatal, so this affects the performance collector shared by the CSM, CSM `0x02`, and CM v2 oracles. Some clients may not support v2 when the release is deployed.
- **Slot duration.** [consensus-specs#4926](https://github.com/ethereum/consensus-specs/pull/4926) renames `SECONDS_PER_SLOT` to `SLOT_DURATION_MS`. The oracle will fail to parse the config of any node that drops the old field.

#### Technical Specification

- For post-Gloas duties, use `GET /eth/v2/validator/duties/proposer/{epoch}`. Its `dependent_root` is `get_block_root_at_slot(state, compute_start_slot_at_epoch(epoch - 1) - 1)` (or the genesis block root on underflow). If a node does not support v2, fall back to v1 and treat a `dependent_root` mismatch as non-fatal. Do not require v2 before Gloas activation.
- Accept either `SLOT_DURATION_MS` or `SECONDS_PER_SLOT`. Raise an error when the two are inconsistent or when the duration is not a whole number of seconds.
- Read the churn parameters for §3 from the same node config.

### 5. On-chain Verifiers changes

#### Scope

- **Core:**
  - `CLValidatorVerifier`, used by `TopUpGateway`.
  - `CLProofVerifier`, used by `ConsolidationGateway` and `PredepositGuarantee`.
  - `ValidatorExitDelayVerifier`.
  - Shared libraries: `contracts/common/lib/CLGIndices.sol` and `GIndex.sol`.
- **Modules:**
  - `src/Verifier.sol`, which verifies validator slashing, withdrawals and balances.
  - Shared libraries: `src/lib/GIndices.sol` and `src/lib/GIndex.sol`.

#### Forward-compatible structures and fixed GIndices

A generalized index identifies a node's path in an SSZ Merkle tree. With ordinary containers and fixed-capacity lists,
increasing the container's tree depth or a list's capacity can move existing nodes. EIP-7688 replaces relevant types
with progressive structures whose existing paths survive compatible extensions:

- `BeaconState` becomes a progressive container. This changes existing field paths once at Gloas; adding fields at new
  positions afterwards does not move existing fields.
- `validators`, `balances` and withdrawals become progressive lists. An item's path depends on its index, rather than a
  fixed maximum list capacity. Growing the list does not move existing items.
- `historical_summaries` remains a fixed-capacity list of `2**24` summaries, and `block_roots` remains an 8192-root
  vector. Their paths from the state root change, but their internal layouts do not.

Both repositories replace deployment-supplied GIs with **Solidity constants**, fixing the paths in contract code.
Deployment sets `GLOAS_SLOT`: proofs for earlier slots use the Electra/Fulu paths, while proofs for that slot and later
use the Gloas paths. Electra and Fulu share the relevant paths.

This removes generic previous/current GI configuration: compatible extensions should not require new GIs. It does not
guarantee compatibility with arbitrary future changes to field positions, types or proof semantics; those still require
code changes.

#### Fields and item paths

ePBS removes `latest_execution_payload_header`, which previously contained the withdrawals root. The modules verifier
now proves withdrawals using `BeaconState.payload_expected_withdrawals`, the list of withdrawals that the execution
payload must honor.

To find an item, the modules verifier joins the path to its list with the path to the item within that list:

```solidity
// Validator i before Gloas
GI_VALIDATORS_PRE_GLOAS.concat(staticListNodeGIndex(i, 40));

// Validator i from Gloas onward
GI_VALIDATORS.concat(progressiveListNodeGIndex(i));
```

Core finds validators the same way after Gloas. Before Gloas, it starts at validator zero and moves to validator `i`
with `GI_FIRST_VALIDATOR_PRE_GLOAS.shr(i)`. Both approaches locate the same validator; they just calculate the path
differently.

`progressiveListNodeGIndex(i)` locates an item in chunks of size `1, 4, 16, 64, ...`. Chunk `k = floor(log4(3*i + 1))`
starts at index `(4**k - 1) / 3`; the helper combines the path to that chunk with the item's offset. There is no single
fixed-depth offset from item zero. For balances, SSZ packs four `uint64` values per node, so the modules verifier
derives the node path from `validatorIndex / 4` and extracts value `validatorIndex % 4`.

#### ePBS: proving blocks missing from EIP-4788

With ePBS, the canonical beacon chain can advance without an execution payload in every beacon block. EIP-4788 is
updated by execution blocks, so it must not be treated as a complete record of canonical beacon block roots, even within
its retention window. Proofs tied to a particular block, such as a withdrawal or a past balance observation, must remain
possible when that block's root is not directly available there.

To prove data from a block whose root is missing, the verifier starts with a later block whose root is available in
EIP-4788. It then proves that the later block's state records the older block's root. This makes the older block's
header trustworthy, so its state root can be used to check the validator, withdrawal or balance proof:

```text
EIP-4788 root
  → recent beacon header
    → recent state root
      → block_roots[targetSlot % 8192]
        OR historical_summaries[summaryIndex].block_summary_root[targetSlot % 8192]
        → target beacon header
          → target state root
            → validator / withdrawal / balance
```

##### Modules: recent and historical block proofs

- `processWithdrawalProof` and `processBalanceProof` now prove the target header through the recent state's
  `block_roots`. The target must be strictly older than the recent block and at most 8192 slots behind it.
- `processHistoricalWithdrawalProof` and `processHistoricalBalanceProof` prove the target header through
  `historical_summaries`. The summary covering the target slot must already have been created.
- Validator and withdrawal/balance witnesses are checked against the **target** header's state root.
- `processSlashedProof` still proves the slashed validator directly against an EIP-4788-authenticated recent header; it
  does not need a separate target block witness.

Each part of the proof uses the paths for the state it reads. For example, when a post-Gloas block is used to prove a
pre-Gloas withdrawal, finding the older block uses Gloas paths. Checking the withdrawal inside that older block uses
pre-Gloas paths.

##### Core: historical fallback for exit-delay proofs

`ValidatorExitDelayVerifier` uses `historical_summaries` for a target block unavailable directly from EIP-4788. It does
not add the modules verifier's `block_roots` route. Consequently, a missing root may require waiting until the next
historical-summary boundary, up to 8192 slots. This delays the exit-delay report and any resulting operator penalty, but
does not make an invalid report valid. This is an accepted temporary trade-off given the planned replacement of this
verifier by EIP-7002-driven exits.

The core base verifiers retain their proof semantics: `CLValidatorVerifier` reconstructs the complete `Validator` leaf,
checks the `parent(slot, proposerIndex)` node and anchors through EIP-4788; `CLProofVerifier` proves the
`parent(pubkeyRoot, withdrawalCredentials)` subtree. Both adopt the new validator GI selection.

#### API and deployment changes

##### Shared configuration

- Remove deployment-supplied GI parameters; constants live in core's `CLGIndices` and modules' `GIndices` libraries.
- Replace `PIVOT_SLOT` / `pivotSlot` with `GLOAS_SLOT` / `gloasSlot` and generic `*_PREV` / `*_CURR` GI getters with
  explicit pre-Gloas/Gloas getters. Deployment remains responsible for choosing the correct network fork slot.
- Proof generators must derive progressive-list paths instead of shifting a first-leaf GI by the item index.

##### Core

- `CLValidatorVerifier` and `CLProofVerifier` constructors take only `gloasSlot` instead of two GIs and a pivot slot.
  The verifier-related constructor inputs of their consumers and `ValidatorExitDelayVerifier` are reduced accordingly.
- All three verifiers expose `GI_FIRST_VALIDATOR_PRE_GLOAS` and `GI_VALIDATORS`. The exit-delay verifier also exposes
  `GI_FIRST_HISTORICAL_SUMMARY_PRE_GLOAS` and `GI_FIRST_HISTORICAL_SUMMARY`.
- Public write-method signatures of `TopUpGateway`, `ConsolidationGateway`, `PredepositGuarantee` and
  `ValidatorExitDelayVerifier` remain unchanged.

##### Modules

- `Verifier` drops the constructor's `GIndices` struct. It retains chain/module settings, including
  `FIRST_SUPPORTED_SLOT`, `CAPELLA_SLOT` and the new `GLOAS_SLOT`, with
  `CAPELLA_SLOT <= FIRST_SUPPORTED_SLOT <= GLOAS_SLOT`.
- Field-root getters are `GI_VALIDATORS`, `GI_BALANCES`, `GI_WITHDRAWALS`, `GI_BLOCK_ROOTS` and
  `GI_HISTORICAL_SUMMARIES`, each with a `*_PRE_GLOAS` counterpart. `GI_BLOCK_ROOT_IN_SUMMARY` identifies the summary's
  block-root vector root, not its first leaf.
- `ProcessWithdrawalInput` now contains both `recentBlock` and `withdrawalBlock` (header plus proof). Both withdrawal
  methods use this input; the separate `ProcessHistoricalWithdrawalInput` type is removed.
- `ProcessBalanceProofInput` adds `balanceBlock` (header plus proof). The historical balance input retains its
  `recentBlock` and `historicalBlock` witnesses. Integrations must update calldata encoding for the changed tuples.
- `bytes32 withdrawalCredentials` / `WITHDRAWAL_CREDENTIALS` replaces the address-only constructor input/getter.
  Withdrawal proofs check the validator's full credentials and the withdrawal recipient against their address portion.

## 6. Protocol parameters changes

**EIP-8061 in Ethereum.** EIP-8061 ("Increase exit and consolidation churn", Glamsterdam / [Gloas]) splits Electra's single balance-based churn function into three separate ones:

- **activation** — still **capped** at `MAX_PER_EPOCH_ACTIVATION_CHURN_LIMIT` = 256 ETH/epoch (**57,600 ETH/day**);
- **exit** — **dynamic cap**, grows with the stake: `exit_churn = max(128, total_active_balance // 2^15)`; at ~43M ≈ **1,312 ETH/epoch** = **295,200 ETH/day**;
- **consolidation** — its own quota `total_active_balance // 2^16`; at ~43M ≈ **656 ETH/epoch** = **147,600 ETH/day**;

### Entry and Exit Churn imbalance problem

With the introduction of EIP-8061, the exit churn can grow significantly while the entry churn remains capped. This opens up a potential attack vector on a permissionless liquid staking protocol like Lido. If the attacker controls a significant portion of the protocol's stake (stETH in Lido's case), they can submit withdrawal requests for the entire stake they hold and stake it all immediately after the withdrawal requests finalize. If the protocol follows the updated limits for exits, the time for the stake to be withdrawn can be significantly shorter than the time for this stake to be re-staked, potentially allowing the attacker to dilute protocol APR, or even put the majority of the protocol's stake into the entry queue.

At the current ETH TVL of the Lido protocol, any meaningful attack of this type will require amounts of stETH above 200k stETH. It is also important to note that this attack comes with no explicit profit for the attacker, and primarily serves to disrupt the protocol's operations and potentially harm other participants. The attacker would also face notable opportunity costs and risks associated with locking up a large amount of stETH, which may deter such behavior.

Despite the low probability of such an attack occurring, it is proposed to limit the amount of ETH VEBO can request for exit within a day to match the network activation churn of **57,600 ETH/day**.

### Proposed limit values

The values for items 1 and 2 are sized for **staking growth up to 60M ETH of active balance**: at that balance they match the network churn allowed by EIP-8061, so the limits do not have to be revisited as the network grows.

1. **`exitedEthAmountPerDayLimit` = 411,975 ETH/day** — the network exit churn at
   a 60M active balance. 

2. **`consolidationEthAmountPerDayLimit` = 205,875 ETH/day** — the network
   consolidation churn at a 60M active balance.

3. **`maxBalanceExitRequestedPerReportInEth` = 11,520 ETH** — reduced from 19,200 ETH based on the note above.

4. **`ConsolidationGateway`** — the limits configured in this service are left
   unchanged at this stage.

### SRv3 vs Glamsterdam (EIP-8061): what changes

The baseline is the **SRv3 ([LIP-35](./lip-35.md))** parameters. SRv3 sanity limits are set in ETH and were tuned for pre-Glamsterdam churn (entry = exit = 256 ETH/epoch).

EIP-8061 changes the **network exit churn** and the **consolidation churn**; entry stays under the activation cap and does not speed up. So here is what should change:

| SRv3 parameter                          | Current | After Glamsterdam      | Why                                                                           |
|-----------------------------------------|---------|------------------------|-------------------------------------------------------------------------------|
| `exitedEthAmountPerDayLimit`            | 57,600  | **~411,975 ETH/day**   | The network releases this much; otherwise a legitimate mass exit hits the cap |
| `maxBalanceExitRequestedPerReportInEth` | 19,200  | **11,520**             | See rationale above                                                           |
| `consolidationEthAmountPerDayLimit`     | 93,375  | **205,875 ETH/day**    | The network consolidation churn, with headroom up to 60M                      |
| `appearedEthAmountPerDayLimit`          | 57,600  | **57,600** (no change) | Activation churn is capped (256), entry does not speed up                     |

## Links

- [Gloas consensus specification](https://github.com/ethereum/consensus-specs/tree/master/specs/gloas)
- EIPs
  - [EIP-7732: Enshrined Proposer-Builder Separation](https://eips.ethereum.org/EIPS/eip-7732)
  - [EIP-8061: Increase exit and consolidation churn](https://eips.ethereum.org/EIPS/eip-8061)
  - [EIP-7917: Deterministic proposer lookahead](https://eips.ethereum.org/EIPS/eip-7917)
  - [EIP-7688: Forward compatible consensus data structures](https://eips.ethereum.org/EIPS/eip-7688)
- [beacon-APIs#563: proposer duties v2](https://github.com/ethereum/beacon-APIs/pull/563)
- [consensus-specs#4926: `SLOT_DURATION_MS`](https://github.com/ethereum/consensus-specs/pull/4926)
- [LIP-35. Staking Router v3](./lip-35.md)

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

