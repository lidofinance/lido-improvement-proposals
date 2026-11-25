---
lip: 44
title: Optimistic Top-Ups for 0x02 Validators
status: WIP
author: Raman Siamionau (@F4ever)
discussions-to: <create a thread on https://research.lido.fi/ and drop the link here>
created: 2026-11-16
updated: 2026-11-24
---

# LIP-44. Optimistic Top-Ups for `0x02` Validators

## Simple Summary

Lido stakes ETH on a `0x02` validator in two steps. The first deposit of 32 ETH, the seed deposit, creates the validator. Later deposits, the top-ups, raise its balance up to 2,048 ETH. Today a top-up waits until the validator is active, so the ETH for it waits in the Lido buffer during the whole deposit queue.

This proposal lets Lido send a top-up soon after the seed deposit, before the validator is active. The top-up goes through the Deposit Security Module ([DSM](https://docs.lido.fi/contracts/deposit-security-module)) and inherits the protections of a seed deposit: the guardians check every new key used for a seed deposit, one guardian can pause all deposits, and ETH limits are shared between seed deposits and top-ups. The proposal removes TopUpGateway and its consensus-layer proofs.

## Abstract

We propose to deploy a new `DepositSecurityModule` with a function `topUpBufferedEther` that sends a top-up to the deposited keys of a staking module without a consensus-layer proof of validator state. Only the holder of the top-up role can call it, and it uses the same pause, block distance, and ETH limit per call as a seed deposit. We propose to upgrade `StakingRouter` and the `0x02` staking modules to support this flow and to remove `TopUpGateway`. The proposal requires [LIP-42](./lip-42.md).

## Motivation

In the flow of [LIP-35](./lip-35.md), `TopUpGateway` accepts a top-up only with a consensus-layer proof that the validator is active. A new key gets its 32 ETH seed deposit and then waits in the deposit queue. The top-up amount sits in the buffer for this whole time. An optimistic top-up sends it to the target key early.

Optimistic top-ups have the following benefits:

- **Allocation stays correct.** The stake goes to its target key, so later designs that depend on the stake distribution (the new VEBO and active rebalancing introduced in [LIP-38](./lip-38.md)) need no special case.
- **The stake reaches its target key early.** The top-up waits at least the block distance after the seed deposit, not the whole deposit queue.
- **The stake always has a destination.** An optimistic top-up does not need free capacity on other keys. It also works when Lido directs stake to a new module or to new operators.
- **Less code.** TopUpGateway and its proofs are removed. The proof code depends on the layout of the beacon state, so each consensus-layer fork needs a review of it.

## Specification

### Overview

An optimistic top-up is safe only if Lido's seed deposit is the first deposit with a valid signature for the key. That deposit fixes the withdrawal credentials. The guardians sign a seed deposit only when its keys are not front-run, and they keep checking every deposited key in each block. The risk that this proposal changes is a front-run that the guardians miss: the key can then receive top-ups.

The top-up depositor sends a top-up through a new DSM function. A top-up goes out only if all of these conditions are true:

- The DSM is not paused.
- The block distance has passed: at least `minDepositBlockDistance` blocks of the module since the last deposit call to the module and since the last deposit call through the DSM. Seed deposits and top-ups count as deposit calls.
- Each key in the call is a deposited key of the module. A key is deposited when the module has returned it for a seed deposit through the DSM.

**Flow**

1. The DSM makes the seed deposit with the guardian signatures, as today. StakingRouter and the DSM store the block as the last deposit block.
2. Each guardian checks the deposited keys in each block, as today. On a front-run, the guardian pauses the DSM.
3. After the block distance, the depositor bot calls `DSM.topUpBufferedEther` through the top-up depositor with the node operator and the index of each key and a limit for each key.
4. The DSM checks the caller, the pause and the block distance, and calls `StakingRouter.topUp`.
5. The module checks that each key is deposited and returns its public key and an amount. StakingRouter deposits the ETH to the returned public keys. StakingRouter and the DSM store the block as the last deposit block.

The module limits each amount by the balance and the status of the key (requirements 4 and 5 in [Staking modules](#staking-modules)). It learns them from consensus-layer proofs: [LIP-42](./lip-42.md#new-balance-tracking-mechanism-for-cmv2) specifies how they update `keyAllocatedBalance`, and [proof-based exit accounting](./lip-42.md#proof-based-exit-accounting) covers exits.

### Rationale

**Why a top-up needs no guardian signature.** The guardians attest to one fact about a key: Lido's seed deposit is its first valid deposit. This fact does not change after the seed deposit, and the guardians keep checking it in each block to catch potential front-running. A signature for each top-up would attest to the same fact again. It would add a new guardian message and a new check in the council daemon, and top-ups would stop when the quorum is offline. Since the seed deposit was already attested, a simple DSM pause in case of a front-run is sufficient to prevent invalid top-ups.

**Why the delay is the block distance.** StakingRouter already stores the last deposit block of each module, and the DSM already reads it. A key of a module becomes deposited only in a seed deposit to that module, so no deposited key of the module is newer than this block.

**Why the DSM is the entry point.** All deposits keep one entry point. The DSM already holds the pause flag and already checks the block distance. StakingRouter keeps one authorized caller for `deposit` and `topUp`.

**Why top-ups use the DSM-wide distance.** The DSM resets the distance on a seed deposit to any module, so colluding guardians cannot deposit to several modules in one window. Top-ups join the same sequence of deposit calls. The ETH that can leave the buffer per block distance is therefore the same with and without top-ups, across all modules (see [Security Considerations](#security-considerations)).

**Why every call uses the distance.** `topUp` stores the last deposit block also when the call deposits no ETH, as `deposit` does when the module returns no keys. One rule for all calls keeps the bound on deposit calls simple. A call with a zero amount still changes the module: CSM removes keys from its top-up queue in it. The depositor bot puts these keys into calls that also deposit ETH, so the cleanup alone rarely uses a slot.

**Why the top-up depositor sets a limit for each key.** The module does not see every balance on the consensus layer. An [EIP-7251](https://eips.ethereum.org/EIPS/eip-7251) consolidation moves the balance of the source validator to the target key, and the module counts this balance only after a balance proof. The depositor bot reads the consensus layer and lowers the limit. The limit can only lower the amount.

**Why the call is permissioned.** The top-up depositor selects the keys and the limits. In CSM the top-up queue removes a key when its allocation equals its limit, so an open call with a zero limit could remove any key from the queue.

### Technical Specification

#### DepositSecurityModule

We propose to deploy a new `DepositSecurityModule`. It is DSM v5 of [LIP-37](./lip-37.md) with the additions below. The guardian set, the quorum, the signed message formats, `pauseDeposits`, `unvetSigningKeys` and `depositBufferedEther` do not change.

```solidity
interface IDepositSecurityModule {
    /// @notice Returns the top-up depositor, the account that can call `topUpBufferedEther`.
    function getTopUpDepositor() external view returns (address);

    /// @notice Sets the top-up depositor.
    /// @dev Only the owner can call it. Reverts if `newValue` is the zero address.
    function setTopUpDepositor(address newValue) external;

    /// @notice Calls `STAKING_ROUTER.topUp` with the given arguments.
    /// @param stakingModuleId The ID of the staking module.
    /// @param keyIndices Indices of the keys within their node operators.
    /// @param operatorIds IDs of the node operators that own the keys.
    /// @param topUpLimits Maximum amount (in wei) that each key can receive.
    /// @dev Reverts if any of the following is true:
    ///   - the caller is not the top-up depositor;
    ///   - deposits are paused;
    ///   - the block distance has not passed.
    function topUpBufferedEther(
        uint256 stakingModuleId,
        uint256[] calldata keyIndices,
        uint256[] calldata operatorIds,
        uint256[] calldata topUpLimits
    ) external;

    event TopUpDepositorChanged(address newValue);

    error NotATopUpDepositor(address caller);
}
```

`topUpBufferedEther` makes these checks in order, calls `STAKING_ROUTER.topUp` with the same arguments and then stores the current block as the DSM-wide last deposit block, as `depositBufferedEther` does:

| # | Check                                                                                                             | Revert                           | Purpose                                                                                     |
|---|-------------------------------------------------------------------------------------------------------------------|----------------------------------|---------------------------------------------------------------------------------------------|
| 1 | `msg.sender` is the top-up depositor                                                                              | `NotATopUpDepositor(msg.sender)` | The top-up depositor selects the keys and the limits, so the call is not open to anyone     |
| 2 | `isDepositsPaused` is false                                                                                       | `DepositsArePaused()`            | One pause for seed deposits and top-ups. One guardian can set it                            |
| 3 | `_isMinDepositDistancePassed(stakingModuleId)`: the larger of the module and DSM-wide last deposit blocks is at least `minDepositBlockDistance` of the module in the past | `DepositTooFrequent()`           | The same check as for a seed deposit. Every deposited key of the module is at least this old |

`VERSION` changes from `5` to `6`, because the interface of the contract changes.

#### StakingRouter

We propose a new `StakingRouter` implementation. `topUp` loses the `_pubkeys` argument.

```solidity
interface IStakingRouter {
  /// @notice Performs top-up calls to the official Deposit contract.
  /// @param _stakingModuleId ID of the staking module to be deposited.
  /// @param _keyIndices List of keys' indices.
  /// @param _operatorIds List of node operator IDs.
  /// @param _topUpLimits Maximum amount (in wei) that can be deposited per key, set by the top-up depositor.
  /// @dev Only the DepositSecurityModule is allowed to call this method.
  function topUp(
    uint256 _stakingModuleId,
    uint256[] calldata _keyIndices,
    uint256[] calldata _operatorIds,
    uint256[] calldata _topUpLimits
  ) external;
}
```

Changes in `topUp`:

1. **Caller.** `topUp` accepts a call only from the address that `LidoLocator.depositSecurityModule()` returns, as `deposit` does.
2. **Last deposit block.** `topUp` stores the current block and timestamp as the last deposit of the module before it calls the module. `depositBufferedEther` already reads the block.
3. **ETH limit of a call.** The amount of a call is at most the smaller of the module allocation and `maxDepositsPerBlock × 32 ETH` of the module. This is the limit of a seed deposit call.
4. **Public keys.** `topUp` does not take public keys from the caller. It calls `allocateDeposits` of the module without them and deposits to the public keys that the module returns (see [Staking modules](#staking-modules)).
5. **Removed.** `maxTopUpPerBlockGwei`, `setMaxTopUpPerBlockGwei`, `getMaxTopUpPerBlockGwei`, the event `MaxTopUpPerBlockGweiSet`, the error `InvalidMaxTopUpPerBlockGwei` and the `_maxTopUpPerBlockGwei` argument of `initialize`.

#### LidoLocator

We propose a new `LidoLocator` implementation. `depositSecurityModule()` returns the new DSM. `topUpGateway()` is removed.

#### TopUpGateway

After the upgrade StakingRouter no longer accepts calls from TopUpGateway, and the contract has no function in the protocol.

The upgrade removes TopUpGateway from CircuitBreaker with `registerPauser(topUpGateway, address(0))`. This removes it from the enumerable set of pausables of CircuitBreaker ([LIP-34](./lip-34.md)).

#### Staking modules

A staking module that receives optimistic top-ups must meet these requirements. The status column describes Curated Module v2 and the `0x02` version of CSM, together with the changes of LIP-42.

| # | Requirement                                                                                                                                                                                  | Status                                                                                                                                                                                                                                                                                             |
|---|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1 | `allocateDeposits` accepts only a deposited key                                                                                                                                              | Exists. Curated Module v2 reverts with `SigningKeysInvalidOffset` when `keyIndex >= totalDepositedKeys`. CSM requires each key to be the head of its top-up queue and reverts with `InvalidTopUpOrder` otherwise. A key enters this queue in a seed deposit. This check is now a security boundary |
| 2 | `allocateDeposits` takes no public keys and returns the public key and the amount of each key from the storage of the module, in the order of the call, also when `maxDepositAmount` is zero | New. Today the modules take the public keys from the caller and compare them with the storage (`SigningKeys.verifySigningKey`), and Curated Module v2 returns an empty array for a zero `maxDepositAmount`                                                                                         |
| 3 | A key becomes deposited only in `obtainDepositData`, which StakingRouter calls from `deposit`. No other function marks a key as deposited                                                    | Exists for the deposit path. Each module must confirm it for its migration and administrative functions                                                                                                                                                                                            |
| 4 | The module limits the amount for a key to the smaller of the limit from the top-up depositor and `2,016 ETH − keyAllocatedBalance`.           | Exists (`capTopUpLimitsByKeyBalance`). In CSM a duplicate key reverts, because only the head of the queue is accepted                                                                                                                                                                              |
| 5 | The module gives a zero amount to a withdrawn, slashed or exited key, without a revert                                                                                                       | Withdrawn and slashed: exists. Exited: [LIP-42](./lip-42.md#proof-based-exit-accounting). The zero amount keeps the cleanup of the CSM top-up queue working                                                                                                                                        |
| 6 | After a partial withdrawal, `keyAllocatedBalance` decreases, so the key can receive top-ups again                                                                                            | Curated Module v2: [LIP-42](./lip-42.md#new-balance-tracking-mechanism-for-cmv2). CSM `0x02`: not needed, because [LIP-38](./lip-38.md) disables partial withdrawals for it                                                                                                                        |

**The new `allocateDeposits`.** The function loses the `pubkeys` argument and returns the public keys with the amounts:

```solidity
interface IStakingModuleV2 {
    /// @notice Allocates a top-up amount to each of the given keys.
    /// @param maxDepositAmount Maximum total amount (in wei) for the call.
    /// @param keyIndices Indices of the keys within their node operators.
    /// @param operatorIds IDs of the node operators that own the keys.
    /// @param topUpLimits Maximum amount (in wei) that each key can receive.
    /// @return pubkeys Public keys of the given keys, from the storage of the module.
    /// @return allocations Amount (in wei) for each key.
    /// @dev Reverts if a key is not a deposited key of the module.
    function allocateDeposits(
        uint256 maxDepositAmount,
        uint256[] calldata keyIndices,
        uint256[] calldata operatorIds,
        uint256[] calldata topUpLimits
    ) external returns (bytes[] memory pubkeys, uint256[] memory allocations);
}
```

The two returned arrays have the same length and the same order as the input, also for a zero `maxDepositAmount`. The module reads each public key from its storage at the node operator and the index of the key.

`keyAllocatedBalance`, the balance of a key above its seed as the module records it, and its update rules are specified in LIP-42. LIP-42 also gives StakingRouter the top-up capacity of a module (`getTopUpCapacity`), which counts keys that are not active yet.

#### Guardian check

The guardians sign nothing for a top-up. In each block the council daemon checks, as today, that the first valid deposit of every deposited key carries Lido's withdrawal credentials, and pauses the DSM if it does not.

#### Off-chain components

**Depositor bot.** It operates the top-up depositor. It waits for the block distance and calls `DSM.topUpBufferedEther` with the node operator and the index of each key. It no longer builds proofs. Before each call it runs its own front-run check of the deposited keys and does not top up a module that fails it. This check is implemented in the depositor bot codebase, separately from the council daemon. It computes each limit from the consensus layer: the target balance minus the current balance, the pending deposits and the pending consolidations to the key. It skips exiting keys and keys with a limit below the minimum top-up.

**Council daemon.** It needs a release that accepts DSM `VERSION` `6` and uses the same signing logic as for `5`.

#### Parameters

| Parameter                                                                        | Value                                                                       | Where                                                   |
|----------------------------------------------------------------------------------|-----------------------------------------------------------------------------|---------------------------------------------------------|
| ETH limit of a top-up call                                                       | `maxDepositsPerBlock × 32 ETH` of the module                                | StakingRouter, existing parameter                       |
| Block distance                                                                   | `minDepositBlockDistance` of the module, values unchanged                   | StakingRouter, existing parameter                       |
| `topUpDepositor`                                                                 | The `DelegationContract` that holds `TOP_UP_ROLE` on TopUpGateway today     | DSM, new parameter                                      |
| Owner of the DSM                                                                 | Aragon Agent                                                                | DSM, unchanged                                          |
| Guardians, quorum, `pauseIntentValidityPeriodBlocks`, `maxOperatorsPerUnvetting` | The values of the current DSM                                               | DSM, unchanged                                          |
| Target balance of a key                                                          | 2,046.75 ETH                                                                | Depositor bot, from `targetBalanceGwei` of TopUpGateway |
| Minimum top-up of a key                                                          | 2 ETH                                                                       | Depositor bot, from `minTopUpGwei` of TopUpGateway      |
| Hard cap for a key                                                               | 2,016 ETH above the seed, for the balance that the module records           | Module constant, unchanged                              |

### Upgradability

| Contract                                        | Change                                                                                                    |
|-------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| `DepositSecurityModule`                         | Not upgradable. Redeployed at a new address with the current guardians, quorum and parameters and the top-up depositor. `STAKING_MODULE_UNVETTING_ROLE` on StakingRouter moves from the old DSM to the new one |
| `StakingRouter`                                 | New implementation: `topUp` as described above                                                            |
| `LidoLocator`                                   | New implementation: the new DSM address, no `topUpGateway`                                                |
| `0x02` staking modules (Curated Module v2, CSM) | New implementations with the new `allocateDeposits`. Same vote as `StakingRouter`, because both use the same interface |
| `TopUpGateway`                                  | Removed from CircuitBreaker and from `LidoLocator`       |

## Security Considerations

### A missed front-run costs more

This is the risk that the proposal changes. If the guardians miss the front-run of a key, Lido loses its 32 ETH seed deposit today. With optimistic top-ups the same key can then receive up to 2,016 ETH of top-ups, the cap of the module. The top-up depositor's target balance does not bound this: a compromised or faulty top-up depositor can send up to the cap. The loss on one key therefore grows from 32 ETH to up to 2,048 ETH.

A top-up to such a key requires both of these to miss the front-run or to be compromised:

- **Every guardian.** The key can receive a top-up only after the block distance since its seed deposit. In this window each guardian checks the deposit log with the seed deposit in it, and one guardian is enough to pause. The check takes the list of deposited keys from the Keys API, so a delay in the Keys API shortens the window.
- **The depositor bot.** It runs its own check before each call. All guardians run the same daemon, so one error in the daemon keeps every guardian silent. The check of the depositor bot is a separate implementation for this reason.

If both fail, the last protection is the Execution Delegation Framework: the owner of the `DelegationContract`, currently a multisig, can revoke the delegate key of the top-up depositor at once, without a DAO vote (see [The top-up depositor](#the-top-up-depositor)).

### No extra leverage for a compromised quorum

Lido already trusts the guardians for every seed deposit. If the quorum is compromised, the attackers take seed deposits until one honest guardian pauses. The loss is bounded by the ETH that can leave the buffer per block distance, multiplied by the time until the pause. Seed deposits and top-ups share the DSM pause, the DSM-wide block distance and the ETH limit per call, so top-ups do not raise this bound. A top-up reaches an attacker's key only after a seed deposit to that key.

### The top-up depositor

The top-up depositor selects the keys and the limits. It cannot redirect ETH: it sends no public keys, and the module returns the public key of a deposited key from its storage. A compromised top-up depositor can:

- send top-ups to an exiting or slashed validator before the module learns about the event;
- lift a key above 2,048 ETH when the balance that the module records is stale;
- use every block distance, which delays seed deposits to all modules;
- remove keys from the CSM top-up queue with zero limits.

The top-up depositor is an Execution Delegation Framework `DelegationContract` ([LIP-37](./lip-37.md)). Its owner can revoke a compromised delegate key at once, without a DAO vote. The DSM pause also stops it.

### Reorgs

A reorg cannot put another deposit before the seed deposit. The seed deposit binds to the deposit root, so it reverts if the deposit log changed after the guardians signed. The key is then not deposited and cannot receive a top-up.

## Known Issues

- **Seed deposits and top-ups share one sequence of deposit calls.** The protocol makes at most one deposit call per block distance, for all modules and both paths together. Today top-ups have a separate protocol-wide cap. A top-up, also one with a zero amount, delays seed deposits to every module, and a top-up can wait longer than the block distance when other deposits use the slots. The depositor bot orders the calls. Governance can change `maxDepositsPerBlock` and `minDepositBlockDistance` of the modules.
- **The module learns about an exit or a slashing only from a proof.** Until the proof of LIP-42 arrives, a top-up can go to an exiting or slashed validator. The ETH is not lost, but it returns only after the validator is withdrawable. The depositor bot skips exiting keys.
- **The cap is as fresh as the last proof.** When the balance that the module records is stale, a top-up can lift a key above 2,048 ETH. The excess is not lost: the withdrawal sweep returns it to Lido.
- **The CircuitBreaker Committee can no longer pause top-ups.** The DSM pause replaces the TopUpGateway pause. One guardian can set it, but it also stops seed deposits, and only the owner of the DSM can remove it with a DAO vote.

## Links

- [LIP-33. Community Staking Module v3 and Curated Module v2](./lip-33.md)
- [LIP-34. CircuitBreaker](./lip-34.md)
- [LIP-35. Staking Router v3](./lip-35.md)
- [LIP-37. Execution Delegation Framework](./lip-37.md)
- [LIP-38. Withdrawals Oracle and Active Rebalancing](./lip-38.md)
- [LIP-42. Curated Module v2 Phase 2. ValMart](./lip-42.md)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

<!-- my unicorn
                           A
    |                     / |
    |\              _____/  /
    | \            /    , \/
    |  \/=========/   _____\
     \   __________  /
       \//        \\/
   /|||/\\|/||/|\\\/|||/\\|/||/|\\\/|||/\\|/||/|\\\
-->
