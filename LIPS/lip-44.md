---
lip: 44
title: Optimistic Top-Ups for 0x02 Validators
status: WIP
author: Raman Siamionau (@F4ever)
discussions-to: <create a thread on https://research.lido.fi/ and drop the link here>
created: 2026-11-16
updated: 2026-11-17
---

# LIP-44. Optimistic Top-Ups for `0x02` Validators

## Simple Summary

Lido stakes ETH on a `0x02` validator in two steps. The first deposit of 32 ETH, the seed deposit, creates the validator. Later deposits, the top-ups, raise its balance up to 2,048 ETH. Today a top-up waits until the validator is active, so the ETH for it waits in the Lido buffer during the whole activation queue.

This proposal lets Lido send a top-up soon after the seed deposit, before the validator is active. The top-up goes through the Deposit Security Module ([DSM](https://docs.lido.fi/contracts/deposit-security-module)) and uses the protections of a seed deposit: the guardians check every new key, and one guardian can pause all deposits. The proposal removes TopUpGateway and its consensus-layer proofs.

## Abstract

We propose to change the top-up flow of [LIP-35](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-35.md) so that a new `0x02` key can receive a top-up before it is active.

We propose to deploy a new `DepositSecurityModule` with one new function, `topUpBufferedEther`. Only the account with the top-up role can call this function, and the call needs no guardian signature. The DSM allows the top-up if it is not paused and if `minDepositBlockDistance` blocks have passed since the last seed deposit or top-up to the staking module.

We propose to upgrade `StakingRouter` so that `topUp` accepts a call only from the DSM, stores the last deposit block of the module, uses the ETH limit of a seed deposit call and takes the public keys from the module. We propose to remove `TopUpGateway` from the protocol, with its consensus-layer proofs, its `LidoLocator` entry and its CircuitBreaker pause.

The staking module accepts a top-up only for a key that it has deposited. The call identifies a key by its node operator and its index, and the module returns the public key from its storage. The module limits the amount for each key from the balance that it stores for the key. The module gives no ETH to a slashed, withdrawn or exited key. The exit proof, the balance tracking with partial withdrawals and the top-up capacity of the module come from [LIP-42](https://github.com/lidofinance/lido-improvement-proposals/pull/139). This proposal requires LIP-42 and adds no proof of its own.

The guardians keep their current duty. They check every deposited key and pause the DSM if they see a front-run. Seed deposits and top-ups to a module share one pause, one block distance and one ETH limit per call.

## Motivation

In the flow of LIP-35, `TopUpGateway` accepts a top-up only with a consensus-layer proof that the validator is active. A new key gets its 32 ETH seed deposit and then waits in the activation queue.

Today a top-up waits until the key is active, so the top-up amount sits in the buffer for one more queue length. An optimistic top-up sends it to the target key early.

Optimistic top-ups have these benefits:

- **Allocation stays correct.** This is the most important point. The stake goes to its target key, so later designs that depend on the stake distribution (VEBO, active rebalancing) need no special case.
- **The stake reaches its target key early.** The minimum wait is `minDepositBlockDistance` of the module.
- **The stake always has a destination.** An optimistic top-up does not need free capacity on other keys. It also works when Lido directs stake to a new module or to new operators.
- **Less code.** TopUpGateway and its proofs are removed. The proof code depends on the layout of the beacon state, so each consensus-layer fork needs a review of it.

## Specification

### Overview

An optimistic top-up is safe only if Lido's seed deposit is the first deposit with a valid signature for the key. That deposit fixes the withdrawal credentials.

The DSM already guarantees this for every seed deposit: the guardians sign only when the keys have valid signatures and are not front-run. The risk is a failure of the DSM itself, for example a compromised quorum.

A delay after the seed deposit gives the guardians time to pause. One honest guardian is enough to pause the DSM. The account with the top-up role sends a top-up through a new DSM function. A top-up to a module goes out only if all of these conditions are true:

- At least `minDepositBlockDistance` blocks have passed since the last seed deposit or top-up to the module.
- The DSM is not paused.
- Each key in the call is a deposited key of the module, also called a used key. A key is deposited when the module has returned it for a seed deposit through the DSM. 

**Synchronous flow**

1. The DSM makes the seed deposit with the guardian signatures, as today. StakingRouter stores the block as the last deposit block of the module.
2. Each guardian checks the deposited keys in each block, as today. On a front-run, the guardian pauses the DSM.
3. After `minDepositBlockDistance` blocks, the depositor bot calls `DSM.topUpBufferedEther` with the node operator and the index of each key and a limit for each key.
4. The DSM checks the caller, the pause and the block distance. It then calls `StakingRouter.topUp`.
5. The module checks that each key is deposited and returns its public key and an amount. StakingRouter deposits the ETH to the returned public keys and stores the current block as the last deposit block of the module.

**Asynchronous flow**

The staking module learns the balance and the status of its validators from consensus-layer proofs. A prover delivers these proofs to the module outside the top-up call. The module uses the stored balance to limit a top-up and the stored status to give no ETH to a slashed, withdrawn or exited key. LIP-42 specifies these proofs, including the exit proof and the partial withdrawal proof.

### Rationale

**Why a top-up needs no guardian signature.** Seed deposits and top-ups to a module share one ETH limit and one block distance. If the guardian quorum is compromised, top-ups therefore add no leverage to the attack (see [Security Considerations](#security-considerations)). A signature for each top-up would add a new guardian message and a new check in the council daemon. Top-ups would also stop when the quorum is offline.

**Why the delay is the block distance of the module.** StakingRouter already stores the last deposit block of each module, and the DSM already reads it. A key of a module becomes deposited only in a seed deposit to that module, so no deposited key of the module is newer than this block.

**Why the DSM is the entry point.** All deposits keep one entry point. The DSM already holds the pause flag and already checks the block distance. StakingRouter keeps one authorized caller for `deposit` and `topUp`.

**Why seed deposits and top-ups share one limit.** A separate limit for top-ups would add to the ETH that can leave the buffer in one block distance. With one limit, the worst case with a compromised quorum is the same with and without top-ups.

**Why the caller keeps a limit for each key.** The module does not see every balance on the consensus layer. An [EIP-7251](https://eips.ethereum.org/EIPS/eip-7251) consolidation moves the balance of the source validator to the target key, and the module counts this balance only after a balance proof. The depositor bot reads the consensus layer and lowers the limit. A limit from the caller can only lower the amount.

**Why the call stays permissioned.** The caller selects the keys and the limits. In CSM the top-up queue removes a key when its allocation equals its limit, so a caller with a low limit can remove a key from the queue. The call is therefore not open to anyone.

### Technical Specification

#### DepositSecurityModule

We propose to deploy a new `DepositSecurityModule`. It is DSM v5 of [LIP-37](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-37.md) with the additions below. The guardian set, the quorum, the signed message formats, `pauseDeposits`, `unvetSigningKeys` and `depositBufferedEther` do not change.

```solidity
interface IDepositSecurityModule {
    /// @notice Returns the account that can call `topUpBufferedEther`.
    function getTopUpDepositor() external view returns (address);

    /// @notice Sets the account that can call `topUpBufferedEther`.
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
    ///   - min deposit distance of the module is not passed.
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

`topUpBufferedEther` makes these checks in order and then calls `STAKING_ROUTER.topUp` with the same arguments:

| # | Check                                                                                                                          | Revert                           | Purpose                                                                                             |
|---|--------------------------------------------------------------------------------------------------------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------|
| 1 | `msg.sender` is the top-up depositor                                                                                           | `NotATopUpDepositor(msg.sender)` | The caller selects the keys and the limit for each key, so the call is not open to anyone           |
| 2 | `isDepositsPaused` is false                                                                                                    | `DepositsArePaused()`            | One pause for seed deposits and top-ups. One guardian can set it                                    |
| 3 | `block.number − getStakingModuleLastDepositBlock(stakingModuleId) >= getStakingModuleMinDepositBlockDistance(stakingModuleId)` | `DepositTooFrequent()`           | Every deposited key of the module is at least this old. This is the reaction time for the guardians |

Other properties of the new DSM:

- **The top-up depositor.** The DSM stores the account with the top-up role as `topUpDepositor`. The constructor takes its initial value as a new argument. Only the owner of the DSM can change it.
- **The DSM-wide distance.** For a seed deposit the DSM also uses its own `lastDepositBlock`, so a seed deposit waits after a seed deposit to any module. `topUpBufferedEther` does not read this value and does not change it.
- **Version.** `VERSION` changes from `5` to `6`, because the interface of the contract changes.
- **Signatures.** The message prefixes include `block.chainid` and the address of the DSM. A guardian signature for the old DSM is therefore not valid for the new DSM.

#### StakingRouter

We propose a new `StakingRouter` implementation. `topUp` loses the `_pubkeys` argument.

```solidity
interface IStakingRouter {
  /// @notice Performs top-up calls to the official Deposit contract.
  /// @param _stakingModuleId Id of the staking module to be deposited.
  /// @param _keyIndices List of keys' indices
  /// @param _operatorIds List of operator indices
  /// @param _topUpLimits Maximum amount (in wei) that can be deposited per key. The caller of the DSM supplies it.
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
2. **Last deposit block.** `topUp` stores the current block and timestamp as the last deposit of the module before it calls the module. It stores them also when the amount of the call is zero. `depositBufferedEther` already reads this value, so a seed deposit to the module also waits `minDepositBlockDistance` blocks after a top-up.
3. **ETH limit of a call.** The amount of a call is at most the smaller of the module allocation and `maxDepositsPerBlock × 32 ETH` of the module. This is the limit of a seed deposit call.
4. **Public keys.** `topUp` does not take public keys from the caller. It calls `allocateDeposits` of the module without them and deposits to the public keys that the module returns (see [Staking modules](#staking-modules)). `topUp` reverts if the module returns a different number of public keys and amounts, or a public key that is not 48 bytes long.
5. **Removed.** `maxTopUpPerBlockGwei`, `setMaxTopUpPerBlockGwei`, `getMaxTopUpPerBlockGwei` and the event `MaxTopUpPerBlockGweiSet`.

These parts of `topUp` do not change: the requirement that the module is active and has `0x02` withdrawal credentials, the checks of the amounts that the module returns, the pull of ETH from Lido and the call with a zero amount that lets CSM advance its top-up queue.

This proposal does not change the allocation between modules. LIP-42 changes it: StakingRouter takes the top-up capacity of a module from the new `getTopUpCapacity` view, deposited keys minus exited keys. This capacity includes the keys that are not active yet, so an optimistic top-up to such a key fits in it.

The limits are separate for each module. A top-up to one `0x02` module does not wait after a seed deposit or a top-up to another module.

#### LidoLocator

We propose a new `LidoLocator` implementation. `depositSecurityModule()` returns the new DSM. `topUpGateway()` is removed.

#### TopUpGateway

After the upgrade no contract of the protocol accepts a call from TopUpGateway. The upgrade unregisters TopUpGateway from CircuitBreaker with `registerPauser(topUpGateway, address(0))`.

These checks and parameters of TopUpGateway have no on-chain replacement:

- The proof of the validator against the [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788) beacon root, with the requirement that the validator is active.
- `maxValidatorsPerTopUp`, `minTopUpBlockDistance` and `maxRootAgeSec`.
- `targetBalanceGwei` and `minTopUpGwei`. The depositor bot applies the target balance when it computes the limit for a key.

#### Staking modules

A staking module that receives optimistic top-ups must meet these requirements. The status column describes Curated Module v2 and the `0x02` version of CSM at staking-modules revision `8df41d65`, together with the changes of LIP-42.

| # | Requirement                                                                                                                               | Status                                                                                                                                                                                                                                   |
|---|-------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1 | `allocateDeposits` accepts only a deposited key                                                                                           | Exists. Curated Module v2 reverts with `SigningKeysInvalidOffset` when `keyIndex >= totalDepositedKeys`. CSM takes keys only from its top-up queue, and a key enters this queue in a seed deposit. This check is now a security boundary |
| 2 | `allocateDeposits` takes no public keys and returns the public key of each key from the storage of the module                             | New. Today the modules take the public keys from the caller and compare them with the storage (`SigningKeys.verifySigningKey`)                                                                                                           |
| 3 | A key becomes deposited only in `obtainDepositData`, which StakingRouter calls from `deposit`. No other function marks a key as deposited | Exists for the deposit path. Each module must confirm it for its migration and administrative functions                                                                                                                                  |
| 4 | The module limits the amount for a key to the smaller of the limit from the caller and `2,016 ETH − keyAllocatedBalance`                  | Exists (`capTopUpLimitsByKeyBalance`)                                                                                                                                                                                                    |
| 5 | The module gives no ETH to a withdrawn, exited or slashed key                                                                             | Specified in LIP-42 (proof-based exit accounting). The module sets the limit to zero for a key with `keyStatus != NONE`. A slashed validator always has an exit epoch, so the exit proof covers it                                       |
| 6 | The module decreases `keyAllocatedBalance` of a key on a partial withdrawal                                                               | Specified in LIP-42 for Curated Module v2 (`processPartialWithdrawalProof`). CSM keeps the balance accounting of LIP-33, where the counter does not decrease. It lets a key receive a top-up again after a partial withdrawal            |

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

The two returned arrays have the same length and the same order. The module reads each public key from its storage at the node operator and the index of the key. The type of `pubkeys` is provisional until the implementation.

**The balance that the module stores.** `keyAllocatedBalance` is the balance of a key above its 32 ETH seed, as the module knows it. A top-up increases the counter at once, and the module caps a top-up so that the counter does not exceed 2,016 ETH. `Verifier.processBalanceProof` is permissionless and can only raise the counter to the proven balance. In Curated Module v2 a partial withdrawal proof sets the counter to the proven balance, also when it is lower (LIP-42). In CSM the counter does not decrease.

The cap therefore bounds the balance of a key as the module records it. It does not bound the total of Lido's deposits to a key over its life: after a partial withdrawal in Curated Module v2 the key can receive top-ups again. The counter can also be too low for a short time. LIP-42 describes this as a known issue: a partial withdrawal proof for a slot before a pending top-up lands on the consensus layer lowers the counter below the real balance, until a newer proof arrives. A partial withdrawal is possible only for an active validator, so a top-up to a key that is not active yet does not add to this window.

**Zero amount, not a revert.** For a slashed, withdrawn or exited key the module returns a zero amount and does not revert. The zero amount keeps the cleanup of the CSM top-up queue working. LIP-42 relies on the same behaviour: with a zero limit CSM removes an exited key from its top-up queue on the next call of the depositor bot. Until then the key holds a place in the queue.

**Dependency on LIP-42.** This proposal adds no proof and no key status to the modules. It uses these parts of [LIP-42](https://github.com/lidofinance/lido-improvement-proposals/pull/139):

- **Proof-based exit accounting.** A permissionless proof that the validator has `exit_epoch != FAR_FUTURE_EPOCH` calls `reportValidatorExited` on the module. The key gets `keyStatus = EXITED`, and its top-up limit becomes zero. The consensus layer never resets an exit epoch, so the status is final.
- **Balance tracking of Curated Module v2.** `processPartialWithdrawalProof` lowers `keyAllocatedBalance` after a partial withdrawal.
- **Top-up capacity.** `getTopUpCapacity` gives StakingRouter the capacity of the module, deposited keys minus exited keys.

Without LIP-42 the module still gives no ETH to a withdrawn key, but it gives ETH to an exited key until the withdrawal proof. The depositor bot is then the only component that skips exited keys.

#### Guardian check

The guardians sign nothing for a top-up. Their duty is the check that the council daemon already runs in each block: for every key that a module has deposited, the first deposit with a valid signature in the deposit contract log carries Lido's withdrawal credentials. If the check fails for a key, the guardian pauses the DSM.

A key that passes this check after its seed deposit is safe for all later top-ups. Under [EIP-6110](https://eips.ethereum.org/EIPS/eip-6110) the consensus layer processes deposits in the order of the deposit contract log, so a later deposit cannot move ahead of the seed deposit.

#### Off-chain components

**Depositor bot.** It waits for the block distance and calls `DSM.topUpBufferedEther` through its `DelegationContract`. It no longer builds proofs and no longer sends public keys: it sends the node operator and the index of each key. It checks the deposit log for a front-run before each call and does not top up a module that fails the check. It computes each limit from the consensus layer: the target balance minus the current balance, the pending deposits and the pending consolidations to the key. It skips exiting keys.

**Prover.** It delivers the balance, slashing, withdrawal, exit and partial withdrawal proofs to the module, with the methods of LIP-42. It must deliver a balance proof when the balance of a key grows by a path that the module does not count, for example a consolidation.

**Council daemon and Data Bus.** No new check and no new message. The daemon must work with the new DSM address. The daemon selects its signing logic by `VERSION` of the DSM and uses the version-5 logic only for the value `5` (`getDsmStrategy`, council daemon `3db0f71`). It therefore needs a release that uses the same logic for version `6`.

**Independent monitoring.** Lido already runs independent deposit monitoring. It gets the front-run check for deposited keys in an implementation separate from the daemon, and it alerts the guardians to pause.

#### Parameters and roles

| Parameter                                                                        | Value                                                                             | Where                                  |
|----------------------------------------------------------------------------------|-----------------------------------------------------------------------------------|----------------------------------------|
| ETH limit of a top-up call                                                       | `maxDepositsPerBlock × 32 ETH` of the module. Module 4: 100 × 32 = 3,200 ETH      | StakingRouter, existing parameter      |
| Block distance                                                                   | `minDepositBlockDistance` of the module, values unchanged. Module 4: 75 blocks    | StakingRouter, existing parameter      |
| `topUpDepositor`                                                                 | The depositor `DelegationContract` that holds `TOP_UP_ROLE` on TopUpGateway today | DSM, new parameter                     |
| Owner of the DSM                                                                 | Aragon Agent                                                                      | DSM, unchanged                         |
| Guardians, quorum, `pauseIntentValidityPeriodBlocks`, `maxOperatorsPerUnvetting` | The values of the current DSM                                                     | DSM, unchanged                         |
| Target balance of a key                                                          | 2,046.75 ETH                                                                      | Depositor bot, moved from TopUpGateway |
| Hard cap for a key                                                               | 2,016 ETH above the seed, for the balance that the module records                 | Module constant, unchanged             |

Removed: `maxTopUpPerBlockGwei` of StakingRouter, and all roles and parameters of TopUpGateway (`TOP_UP_ROLE`, `PAUSE_ROLE`, `RESUME_ROLE`, `MANAGE_LIMITS_ROLE`, `maxValidatorsPerTopUp`, `minTopUpBlockDistance`, `maxRootAgeSec`, `targetBalanceGwei`, `minTopUpGwei`).

#### Deployment and rollout

1. Release the depositor bot, the council daemon and the prover. Add the front-run check to the independent monitoring.
2. Deploy the `StakingRouter` implementation, the `LidoLocator` implementation, the staking module implementations and the new DSM. The new DSM gets the current guardians, quorum and parameters and the top-up depositor. Its ownership goes to the Aragon Agent.
3. One DAO vote upgrades `StakingRouter`, `LidoLocator` and the `0x02` staking modules, moves the roles of the old DSM to the new DSM (`STAKING_MODULE_UNVETTING_ROLE` on StakingRouter) and unregisters TopUpGateway from CircuitBreaker.
4. After the vote the guardians sign for the new DSM.

The module upgrade with the new `allocateDeposits` must ship in the same vote as the `StakingRouter` upgrade, because the two contracts must use the same interface. This proposal requires LIP-42. The module release of LIP-42 ships before this vote or in it.

**Redeployed at a new address:**

| Contract                | Repointing required                                                                              |
|-------------------------|--------------------------------------------------------------------------------------------------|
| `DepositSecurityModule` | New `LidoLocator` implementation that points at the new DSM. The roles of the old DSM move to it |

**New implementation:**

| Contract                                        | Notes                                                                |
|-------------------------------------------------|----------------------------------------------------------------------|
| `StakingRouter`                                 | `topUp` changes as described above                                   |
| `LidoLocator`                                   | New DSM address, no `topUpGateway`                                   |
| `0x02` staking modules (Curated Module v2, CSM) | New `allocateDeposits`. Same vote as `StakingRouter`                 |
| Staking modules and verifiers of LIP-42         | Prerequisite, not a part of this proposal. Before this vote or in it |

**Removed from the protocol:**

| Contract       | Notes                                                                                 |
|----------------|---------------------------------------------------------------------------------------|
| `TopUpGateway` | StakingRouter no longer accepts its calls. Its CircuitBreaker registration is removed |

## Security Considerations

### The security assumptions do not change

Lido already trusts the guardians for every seed deposit. If the guardian quorum is compromised, the attackers take seed deposits until one honest guardian pauses the protocol. The proof on the top-up path does not prevent this. It protects one path while the path next to it rests on the guardians.

In both cases the loss has the same bound: the ETH that can leave the buffer per block distance, multiplied by the time until one honest guardian pauses. The design keeps this bound:

- Seed deposits and top-ups share one ETH limit per call and one block distance.
- One guardian can pause, and the pause stops both paths.
- The ETH in the buffer is the upper bound of any loss.

This is the key property. Seed deposits and top-ups use the same limit, so top-ups add no leverage when the quorum is compromised. A guardian signature for a top-up is therefore not necessary. It is enough that a top-up goes only to a key that the module marks as deposited. This keeps the level of trust of today.

A top-up is not an easier path for an attacker. A top-up can reach an attacker's key only after a seed deposit to that key. Only the account with the top-up role can send a top-up.

The top-up role cannot redirect ETH. The caller sends no public keys: it selects a key by its node operator and its index, and the module returns the public key from its storage. A top-up can go only to a key that the module has deposited. A compromised account with the top-up role therefore cannot send ETH to a key that did not receive a seed deposit through the guardians.

### What does change

- **The loss on one missed key has a higher bound.** Today, if the guardians miss the front-run of one key, the loss on that key is 32 ETH. With optimistic top-ups, the same key can then receive up to 2,014.75 ETH of top-ups. If the check fails for every key, an attacker takes seed deposits at the same rate, with or without top-ups.

  Before: a loss on the top-up path is possible only if there is a bug in the TopUpGateway proof flow.

- **The top-up role has more trust.** The limit for a key is the smaller of two values:
  - The limit from the caller. The depositor bot computes it off-chain from the consensus layer: the target balance minus the current balance, the pending deposits and the pending consolidations to the key. No proof bounds this value, so the trust in the caller grows here.
  - `2,048 ETH − balance that the staking module records for the key`. In the module this is `2,016 ETH − keyAllocatedBalance` (see [Staking modules](#staking-modules)). A top-up raises the counter at once. When the balance of a validator grows, a prover delivers the new balance to the module (`Verifier.processBalanceProof`). In Curated Module v2 a partial withdrawal proof can lower the counter (LIP-42).

  Before: TopUpGateway required a fresh proof of the validator for every call. Now a top-up can be larger than the free capacity of the key if the balance proof in the module is not fresh, or for a short time after an old partial withdrawal proof (the known issue of LIP-42).

- **Top-ups can go to exiting validators.** The module gives no ETH to an exiting key only after it learns about the event. Until then the bot can top up a validator that is exiting or slashed. The ETH is not lost, but it returns only after the validator is withdrawable. The exit proof of LIP-42 and the check in the depositor bot reduce this window (see [Staking modules](#staking-modules)).

  Before: TopUpGateway required a fresh proof of the validator and checked that the validator is not exiting.

- **The CircuitBreaker Committee can no longer pause top-ups.** The DSM pause replaces the TopUpGateway pause.

  Before: a dedicated committee can pause top-ups through CircuitBreaker ([LIP-34](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-34.md)).

### Mitigations and other risks

**Reaction window.** A key can receive a top-up only `minDepositBlockDistance` blocks after its seed deposit. In this window each guardian checks the deposit log with the seed deposit in it. A wrong seed deposit can be found here, before a top-up goes to the same key. The check takes the list of deposited keys from the Keys API, so a delay in the Keys API shortens the window.

**More than one checker.** All guardians run the same daemon, so an error in the check keeps every guardian silent. The depositor bot runs the same check before each call and does not top up a module that fails it. Lido already runs independent deposit monitoring. It gets the same check in a separate implementation and alerts the guardians to pause.

**Hard cap for each key.** The module never lets the balance that it records for a key grow above 2,016 ETH over the seed, whatever the caller sends. The cap is as fresh as the last proof for the key.

**Reorgs.** A reorg cannot put another deposit before the seed deposit. The seed deposit binds to the deposit root, so it reverts if the deposit log changed after the guardians signed. The key is then not deposited and cannot receive a top-up.

**Compromised top-up role.** The account with the top-up role is an Execution Delegation Framework `DelegationContract` (LIP-37). The owner of this contract can revoke a compromised delegate key at once, without a DAO vote. A new delegate key becomes active after the cooldown of the contract. The DSM pause also stops the account.

### Trust model

| Question                                           | Today                                                                               | New approach                                                                                                                                                                                 |
|----------------------------------------------------|-------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Who confirms that a key is safe for a seed deposit | The guardian quorum, with a signature                                               | No change                                                                                                                                                                                    |
| What confirms that a key is safe for a top-up      | A consensus-layer proof that the validator is active                                | The key is a deposited key of the module, the block distance has passed and no guardian has paused                                                                                           |
| Who can send a top-up                              | The account with `TOP_UP_ROLE` on TopUpGateway                                      | The account with the top-up role in the DSM. It is the same `DelegationContract` as today. The owner of the DSM sets the account, and the owner of the `DelegationContract` replaces its key |
| Who selects the keys and the limits                | The same account, within the limits of the proof                                    | The same account. The module caps the amount                                                                                                                                                 |
| Who can stop top-ups                               | The CircuitBreaker Committee, a separate pause                                      | One guardian, with the DSM pause. The pause also stops seed deposits                                                                                                                         |
| Limits                                             | Separate limits: 3,200 ETH of seed deposits plus 3,200 ETH of top-ups per 75 blocks | One limit: 3,200 ETH per 75 blocks for both paths together. The total limit is two times lower                                                                                               |

## Failure Modes

| Failure                                                                                                                                  | Effect                                                                                                                                                                                                                                                                                                       | Detection and response                                                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| The guardians do not detect the front-run of a key, for example because of an error in the council daemon                                | The key can receive top-ups. The loss bound for the key grows from 32 ETH to 32 ETH plus up to 2,014.75 ETH                                                                                                                                                                                                  | The depositor bot and the independent monitoring run the check in separate implementations. Any guardian then pauses the DSM                           |
| The Keys API is late                                                                                                                     | The reaction window is shorter than the block distance                                                                                                                                                                                                                                                       | Governance keeps `minDepositBlockDistance` of a `0x02` module long enough for a guardian to detect a front-run and pause, with a margin for this delay |
| The account with the top-up role is compromised                                                                                          | The attacker selects deposited keys and limits. The attacker can send top-ups to exiting validators, lift a key above 2,048 ETH when the balance in the module is stale and use every block distance, which delays seed deposits to the module. The attacker cannot send ETH to a key without a seed deposit | The owner of the `DelegationContract` revokes the delegate key. A guardian can pause the DSM                                                           |
| The balance in the module is stale, or too low after an old partial withdrawal proof (LIP-42), and the limit from the caller is too high | A top-up lifts the balance of the key above 2,048 ETH. The excess is not lost: the withdrawal sweep returns it to Lido                                                                                                                                                                                       | Anyone can deliver a balance proof to the module. The depositor bot lowers the limit from consensus-layer data                                         |
| A top-up goes to an exiting or slashed validator before the module learns about the event                                                | The ETH is not lost. It returns only after the validator is withdrawable                                                                                                                                                                                                                                     | The depositor bot skips exiting keys. The prover delivers the exit proof of LIP-42                                                                     |
| The DSM is paused                                                                                                                        | Seed deposits and top-ups stop until the owner of the DSM removes the pause with a DAO vote. Today top-ups continue during a DSM pause                                                                                                                                                                       | Expected behaviour                                                                                                                                     |
| The depositor bot is offline                                                                                                             | No top-ups go out. The ETH stays in the buffer. No loss                                                                                                                                                                                                                                                      | No protocol response is necessary. Top-ups continue when the bot is back                                                                               |
| Seed deposits and top-ups to a module compete for the block distance                                                                     | The module gets at most one call per block distance. For module 4 this is about 96 calls and at most 307,200 ETH per day for both paths together                                                                                                                                                             | The depositor bot orders its calls. Governance can change `maxDepositsPerBlock` and `minDepositBlockDistance` of the module                            |

## Links

- [LIP-33. Community Staking Module v3 and Curated Module v2](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-33.md)
- [LIP-34. CircuitBreaker](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-34.md)
- [LIP-35. Staking Router v3](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-35.md)
- [LIP-42. Curated Module v2 Phase 2. ValMart](https://github.com/lidofinance/lido-improvement-proposals/pull/139) (pull request #139)
- [LIP-37. Execution Delegation Framework](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-37.md)
- Core repository (`lidofinance/core`, contains `DepositSecurityModule`, `StakingRouter` and `TopUpGateway`): https://github.com/lidofinance/core — the code references of this proposal use release [`v4.1.0`](https://github.com/lidofinance/core/tree/v4.1.0)
- Staking modules (`lidofinance/staking-modules`): https://github.com/lidofinance/staking-modules — revision [`8df41d65`](https://github.com/lidofinance/staking-modules/tree/8df41d65)
- Council daemon (`lidofinance/lido-council-daemon`): https://github.com/lidofinance/lido-council-daemon — revision [`3db0f71`](https://github.com/lidofinance/lido-council-daemon/tree/3db0f71)
- Depositor bot (`lidofinance/depositor-bot`): https://github.com/lidofinance/depositor-bot

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

<!-- my python
-. .-.   .-. .-.   .-. .-.   .
  \   \ /   \   \ /   \   \ /
 / \   \   / \   \   / \   \
~   `-~ `-`   `-~ `-`   `-~ `-
-->

