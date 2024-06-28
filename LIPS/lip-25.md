---
lip: 25
title: Staking Router 2.0
status: Proposed
author: Kirill Minenko, Alexander Lukin
discussions-to: https://research.lido.fi/t/lip-25-staking-router-v2-0/7773
created: 2024-05-06
updated: 2024-06-28
---

# LIP-25. Staking Router 2.0

## Simple Summary

This proposal introduces Staking Router v2.0 — the upgrade that should be made to support permissionless modules (such as [Community Staking Module](https://research.lido.fi/t/community-staking-module/5917)) and improve security and scalability of the existing [Staking Router](https://research.lido.fi/t/lip-20-staking-router/3790).

The upgrade focuses on enhancing the Deposit Security Module (DSM), the Validator Exit Bus Oracle (VEBO), the third phase of the Lido Oracle, and reward distribution mechanisms in curated-based modules.

## Motivation

In the current implementation of the Curated modules, key vetting (a process of making keys depositable) occurs through the DAO, and in particular EasyTrack motion, which the operator initiates after submitting keys. DSM changes aim to improve the process of key vetting to be able to work without the governance approval and to accommodate future permissionless modules.

The current VEBO mechanism only requests validators to exit in response to user withdrawal requests. This limitation hinders the protocol's ability to manage validator exits proactively, especially for permissionless modules. From the Staking Router's side, it is proposed to consider the module's share when prioritizing validators for exit.

The introduction of the Community Staking Module, which does not limit the number of node operators, necessitates a scalable solution for the Oracle's third-phase reporting.

Current reward distribution mechanisms, tied to the third-phase finalization hook, risk exceeding block gas limits and complicate the reporting process.

## Specification

### 1. `IStakingModule` interface changes

To support the new [unvetting keys process](https://hackmd.io/@lido/rJrTnEc2a#Unvetting) in the Deposit Security Module and [Boosted Exit Requests](https://hackmd.io/@lido/BJXRTxMRp#Boosted-Exit-Requests1) in the Validator Exit Bus Oracle, the `IStakingModule` interface should endure a number of changes.

All the code in this interface assumes the Solidity v0.8.9 syntax.

#### 1.1. New external methods

As the role of decreasing the number of vetted keys is [proposed to be granted](https://hackmd.io/@lido/rJrTnEc2a#Unvetting) to the Deposit Security Module, the new `decreaseVettedSigningKeysCount` method should be added to the `IStakingModule` interface. Keys unvetting occurs through the `DepositSecurityModule ` contract, which should call the `decreaseVettedSigningKeysCount` method on the module side through the Staking Router.

```solidity
/// @notice Called by StakingRouter to decrease the number of vetted keys for node operator with given id
/// @param _nodeOperatorIds bytes packed array of the node operators id
/// @param _vettedSigningKeysCounts bytes packed array of the new number of vetted keys for the node operators
function decreaseVettedSigningKeysCount(
    bytes calldata _nodeOperatorIds,
    bytes calldata _vettedSigningKeysCounts
) external;
```

#### 1.2. Changes in existing methods

It is proposed to allow modules to signal the Validator Exit Bus Oracle about the necessity of sending a request to exit validators at a specific operator without being tied to demand in the Withdrawal Queue (WQ). For this, it is suggested to modify the current `targetLimit` instrument by adding an additional boosted exit mode, namely, the parameter `bool isTargetLimitActive` is proposed to be replaced with `uint8 targetLimitMode` taking one of the following values:
- `0` – Disabled;
- `1` – Limited stake, smooth exit mode;
- `2` – Limited stake, boosted exit mode.

**Disabled.** This mode implies no limitation. The operator is not restricted in receiving new stakes and does not have additional priorities when choosing validators for exit.

**Smooth exit mode.** The operator has a limit on the number of active validators. As long as the number of active validators of the operator does not exceed the `targetLimit`, the operator receives stakes under general conditions. If this value is reached, the operator stops receiving new stakes (should be implemented at the module level). If the number of active keys of the operator exceeds the `targetLimit`, then such an operator's validators are prioritized for exit in the amount of targeted validators to exit.

**Boosted exit mode.** Similar to smooth mode, but does not consider demand in WQ. The operator's validators in the amount of targeted validators to exit are prioritized for exit and requested without considering demand in WQ.

More details about this change can be found in the [VEBO Improvements specification](https://hackmd.io/@lido/BJXRTxMRp).

For this improvement, the following changes should be made in the existing `IStakingModule` interface methods:
- The `isTargetLimitActive` boolean value of the `getNodeOperatorSummary` method should be replaced with the new `targetLimitMode` uint256 value;
- The `_isTargetLimitActive` boolean param of the `updateTargetValidatorsLimits` method should be replaced with the  `_targetLimitMode` param.

```solidity
/// @notice Returns all-validators summary belonging to the node operator with the given id
/// @param _nodeOperatorId id of the operator to return report for
/// @return targetLimitMode shows whether the current target limit applied to the node operator (0 = disabled, 1 = soft mode, 2 = forced mode)
/// @return targetValidatorsCount relative target active validators limit for operator
/// @return stuckValidatorsCount number of validators with an expired request to exit time
/// @return refundedValidatorsCount number of validators that can't be withdrawn, but deposit
///     costs were compensated to the Lido by the node operator
/// @return stuckPenaltyEndTimestamp time when the penalty for stuck validators stops applying
///     to node operator rewards
/// @return totalExitedValidators total number of validators in the EXITED state
///     on the Consensus Layer. This value can't decrease in normal conditions
/// @return totalDepositedValidators total number of validators deposited via the official
///     Deposit Contract. This value is a cumulative counter: even when the validator goes into
///     EXITED state this counter is not decreasing
/// @return depositableValidatorsCount number of validators in the set available for deposit
function getNodeOperatorSummary(uint256 _nodeOperatorId) external view returns (
    uint256 targetLimitMode,
    uint256 targetValidatorsCount,
    uint256 stuckValidatorsCount,
    uint256 refundedValidatorsCount,
    uint256 stuckPenaltyEndTimestamp,
    uint256 totalExitedValidators,
    uint256 totalDepositedValidators,
    uint256 depositableValidatorsCount
);

/// @notice Updates the limit of the validators that can be used for deposit
/// @param _nodeOperatorId Id of the node operator
/// @param _targetLimitMode taret limit mode
/// @param _targetLimit Target limit of the node operator
function updateTargetValidatorsLimits(
    uint256 _nodeOperatorId,
    uint256 _targetLimitMode,
    uint256 _targetLimit
) external;
```

##### 1.2.1. Backward compatibility notes

The change in the `updateTargetValidatorsLimits` method does not break backward compatibility with the EasyTrack factory `UpdateTargetValidatorLimits` which is used to set limits in the Simple DVT module.

The change from `bool isTargetLimitActive` to `uint8 targetLimitMode` affects the response from the view method `getNodeOperatorSummary`, which may be used in external integrations and off-chain tooling. Tests show that [backward compatibility remains](https://github.com/lidofinance/sr-1.5-compatibility-tests). All modes other than disabled are interpreted by the decoder based on the old interface as the enabled `targetLimit` mode.

#### 1.3. New events

Two new events should be added to the `IStakingModule` interface. The purpose of the new events is to help Council Daemon [find duplicate keys in the deposit message](https://hackmd.io/@lido/rJrTnEc2a#Duplicate-Check).

```solidity
/// @dev Event to be emitted when a signing key is added to the StakingModule
event SigningKeyAdded(uint256 indexed nodeOperatorId, bytes pubkey);

/// @dev Event to be emitted when a signing key is removed from the StakingModule
event SigningKeyRemoved(uint256 indexed nodeOperatorId, bytes pubkey);
```

### 2. `StakingRouter` contract changes

All the code of this contract assumes the Solidity v0.8.9 syntax.

#### 2.1. New Target Share engine for validator exits

When validators are exited from one module, its share decreases, while the shares of other modules increase. This can lead to a situation where a module's share significantly exceeds its target share. To solve this problem Staking Router should consider the module's share when prioritizing validators for exit. It is proposed to represent the target share of a module as a range of values: `stakeShareLimit` and `priorityExitShareThreshold`, where `stakeShareLimit <= priorityExitShareThreshold`.

The lower value `stakeShareLimit` represents the maximum share that can be allocated to a module when distributing stakes among modules. This parameter is the same as the current `targetShare`.

The higher value `priorityExitShareThreshold` represents the module's share threshold, upon crossing which, exits of validators from the module will be prioritized.

These two values allow the module to be in one of three states at any given time:

*Module has not reached share limit*

`currentShare < stakeShareLimit`. The proposed design makes no changes for this state. Everything remains as it is:

- The staking router **prioritizes stake allocation** to modules in this state;
- Validators in a module with this state **do not have any extra priority for exit**.

*Module has reached stake share limit*

`stakeShareLimit <= currentShare <= priorityExitShareThreshold`. The proposed design makes no changes for this state. Everything remains as it is:

- The staking router **does not allocate stake** to modules in this state;
- Validators in a module with this state **do not have any extra priority for exit**.

*Module exceeds priority exit threshold*

`priorityExitShareThreshold < currentShare`. It is the state that is affected by the proposed changes.

- The staking router **does not allocate stake** to modules in this state;
- Validators in a module with this state have an **increased exit priority**.

More details about this change can be found in the [VEBO Improvements specification](https://hackmd.io/@lido/BJXRTxMRp#Target-Share1).

Support of the new Target Share engine in the `StakingRouter` contract requires the following changes:
- Existing external `addStakingModule` method should be updated;
- Existing external `updateStakingModule` method should be updated;
- New internal `_updateStakingModule` method should be created;
- Three new `priorityExitShareThreshold`, `maxDepositsPerBlock` and `minDepositBlockDistance` fields should be added to the `StakingModule` structure. The existing `targetShare` field should be renamed to `stakeShareLimit`;
- Signature of the `StakingModuleShareLimitSet` event (former `StakingModuleTargetShareSet`) should be changed;
- Two new `StakingModuleMaxDepositsPerBlockSet` and `StakingModuleMinDepositBlockDistanceSet` events should be added;
- Five new error types should be created: `ZeroAddressStakingModule`, `InvalidStakeShareLimit`, `InvalidFeeSum`, `InvalidPriorityExitShareThreshold` and `InvalidMinDepositBlockDistance`. Existing `ValueOver100Percent` error type should be removed;
- All methods that interact with the `stakeShareLimit` field should be updated.

```solidity
event StakingModuleShareLimitSet(
    uint256 indexed stakingModuleId,
    uint256 stakeShareLimit,
    uint256 priorityExitShareThreshold,
    address setBy
);

event StakingModuleMaxDepositsPerBlockSet(
    uint256 indexed stakingModuleId,
    uint256 maxDepositsPerBlock,
    address setBy
);

event StakingModuleMinDepositBlockDistanceSet(
    uint256 indexed stakingModuleId,
    uint256 minDepositBlockDistance,
    address setBy
);

error ZeroAddressStakingModule();
error InvalidStakeShareLimit();
error InvalidFeeSum();
error InvalidPriorityExitShareThreshold();
error InvalidMinDepositBlockDistance();

struct StakingModule {
    /// @notice unique id of the staking module
    uint24 id;
    /// @notice address of staking module
    address stakingModuleAddress;
    /// @notice part of the fee taken from staking rewards that goes to the staking module
    uint16 stakingModuleFee;
    /// @notice part of the fee taken from staking rewards that goes to the treasury
    uint16 treasuryFee;
    /// @notice maximum stake share that can be allocated to a module, in BP
    uint16 stakeShareLimit; // formerly known as `targetShare`
    /// @notice staking module status if staking module can not accept the deposits or can participate in further reward distribution
    uint8 status;
    /// @notice name of staking module
    string name;
    /// @notice block.timestamp of the last deposit of the staking module
    /// @dev NB: lastDepositAt gets updated even if the deposit value was 0 and no actual deposit happened
    uint64 lastDepositAt;
    /// @notice block.number of the last deposit of the staking module
    /// @dev NB: lastDepositBlock gets updated even if the deposit value was 0 and no actual deposit happened
    uint256 lastDepositBlock;
    /// @notice number of exited validators
    uint256 exitedValidatorsCount;
    /// @notice module's share threshold, upon crossing which, exits of validators from the module will be prioritized, in BP
    uint16 priorityExitShareThreshold;
    /// @notice the maximum number of validators that can be deposited in a single block
    /// @dev must be harmonized with `OracleReportSanityChecker.churnValidatorsPerDayLimit`
    /// (see docs for the `OracleReportSanityChecker.setChurnValidatorsPerDayLimit` function)
    uint64 maxDepositsPerBlock;
    /// @notice the minimum distance between deposits in blocks
    /// @dev must be harmonized with `OracleReportSanityChecker.churnValidatorsPerDayLimit`
    /// (see docs for the `OracleReportSanityChecker.setChurnValidatorsPerDayLimit` function)
    uint64 minDepositBlockDistance;
}

/**
* @notice register a new staking module
* @param _name name of staking module
* @param _stakingModuleAddress address of staking module
* @param _stakeShareLimit maximum share that can be allocated to a module
* @param _priorityExitShareThreshold module's proirity exit share threshold
* @param _stakingModuleFee fee of the staking module taken from the consensus layer rewards
* @param _treasuryFee treasury fee
* @param _maxDepositsPerBlock the maximum number of validators that can be deposited in a single block
* @param _minDepositBlockDistance the minimum distance between deposits in blocks
*/
function addStakingModule(
    string calldata _name,
    address _stakingModuleAddress,
    uint256 _stakeShareLimit,
    uint256 _priorityExitShareThreshold,
    uint256 _stakingModuleFee,
    uint256 _treasuryFee,
    uint256 _maxDepositsPerBlock,
    uint256 _minDepositBlockDistance
) external onlyRole(STAKING_MODULE_MANAGE_ROLE) {
    if (_stakingModuleAddress == address(0)) revert ZeroAddressStakingModule();
    if (bytes(_name).length == 0 || bytes(_name).length > MAX_STAKING_MODULE_NAME_LENGTH)
        revert StakingModuleWrongName();

    uint256 newStakingModuleIndex = getStakingModulesCount();

    if (newStakingModuleIndex >= MAX_STAKING_MODULES_COUNT) revert StakingModulesLimitExceeded();

    for (uint256 i; i < newStakingModuleIndex; ) {
        if (_stakingModuleAddress == _getStakingModuleByIndex(i).stakingModuleAddress)
            revert StakingModuleAddressExists();
        unchecked {
            ++i;
        }
    }

    StakingModule storage newStakingModule = _getStakingModuleByIndex(newStakingModuleIndex);
    uint24 newStakingModuleId = uint24(LAST_STAKING_MODULE_ID_POSITION.getStorageUint256()) + 1;

    newStakingModule.id = newStakingModuleId;
    newStakingModule.name = _name;
    newStakingModule.stakingModuleAddress = _stakingModuleAddress;
    /// @dev since `enum` is `uint8` by nature, so the `status` is stored as `uint8` to avoid
    ///      possible problems when upgrading. But for human readability, we use `enum` as
    ///      function parameter type. More about conversion in the docs
    ///      https://docs.soliditylang.org/en/v0.8.17/types.html#enums
    newStakingModule.status = uint8(StakingModuleStatus.Active);

    /// @dev  Simulate zero value deposit to prevent real deposits into the new StakingModule via
    ///       DepositSecurityModule just after the addition.
    newStakingModule.lastDepositAt = uint64(block.timestamp);
    newStakingModule.lastDepositBlock = block.number;
    emit StakingRouterETHDeposited(newStakingModuleId, 0);

    _setStakingModuleIndexById(newStakingModuleId, newStakingModuleIndex);
    LAST_STAKING_MODULE_ID_POSITION.setStorageUint256(newStakingModuleId);
    STAKING_MODULES_COUNT_POSITION.setStorageUint256(newStakingModuleIndex + 1);

    emit StakingModuleAdded(newStakingModuleId, _stakingModuleAddress, _name, msg.sender);
    _updateStakingModule(
        newStakingModule,
        newStakingModuleId,
        _stakeShareLimit,
        _priorityExitShareThreshold,
        _stakingModuleFee,
        _treasuryFee,
        _maxDepositsPerBlock,
        _minDepositBlockDistance
    );
}

/**
* @notice Update staking module params
* @param _stakingModuleId staking module id
* @param _stakeShareLimit target total stake share
* @param _priorityExitShareThreshold module's proirity exit share threshold
* @param _stakingModuleFee fee of the staking module taken from the consensus layer rewards
* @param _treasuryFee treasury fee
* @param _maxDepositsPerBlock the maximum number of validators that can be deposited in a single block
* @param _minDepositBlockDistance the minimum distance between deposits in blocks
*/
function updateStakingModule(
    uint256 _stakingModuleId,
    uint256 _stakeShareLimit,
    uint256 _priorityExitShareThreshold,
    uint256 _stakingModuleFee,
    uint256 _treasuryFee,
    uint256 _maxDepositsPerBlock,
    uint256 _minDepositBlockDistance
) external onlyRole(STAKING_MODULE_MANAGE_ROLE) {
    StakingModule storage stakingModule = _getStakingModuleById(_stakingModuleId);
    _updateStakingModule(
        stakingModule,
        _stakingModuleId,
        _stakeShareLimit,
        _priorityExitShareThreshold,
        _stakingModuleFee,
        _treasuryFee,
        _maxDepositsPerBlock,
        _minDepositBlockDistance
    );
}

function _updateStakingModule(
    StakingModule storage stakingModule,
    uint256 _stakingModuleId,
    uint256 _stakeShareLimit,
    uint256 _priorityExitShareThreshold,
    uint256 _stakingModuleFee,
    uint256 _treasuryFee,
    uint256 _maxDepositsPerBlock,
    uint256 _minDepositBlockDistance
) internal {
    if (_stakeShareLimit > TOTAL_BASIS_POINTS) revert InvalidStakeShareLimit();
    if (_priorityExitShareThreshold > TOTAL_BASIS_POINTS) revert InvalidPriorityExitShareThreshold();
    if (_stakeShareLimit > _priorityExitShareThreshold) revert InvalidPriorityExitShareThreshold();
    if (_stakingModuleFee + _treasuryFee > TOTAL_BASIS_POINTS) revert InvalidFeeSum();
    if (_minDepositBlockDistance == 0) revert InvalidMinDepositBlockDistance();

    stakingModule.stakeShareLimit = uint16(_stakeShareLimit);
    stakingModule.priorityExitShareThreshold = uint16(_priorityExitShareThreshold);
    stakingModule.treasuryFee = uint16(_treasuryFee);
    stakingModule.stakingModuleFee = uint16(_stakingModuleFee);
    stakingModule.maxDepositsPerBlock = uint64(_maxDepositsPerBlock);
    stakingModule.minDepositBlockDistance = uint64(_minDepositBlockDistance);

    emit StakingModuleShareLimitSet(_stakingModuleId, _stakeShareLimit, _priorityExitShareThreshold, msg.sender);
    emit StakingModuleFeesSet(_stakingModuleId, _stakingModuleFee, _treasuryFee, msg.sender);
    emit StakingModuleMaxDepositsPerBlockSet(_stakingModuleId, _maxDepositsPerBlock, msg.sender);
    emit StakingModuleMinDepositBlockDistanceSet(_stakingModuleId, _minDepositBlockDistance, msg.sender);
}
```

##### 2.1.1. Backward compatibility notes

The change to the `StakingModule` struct affects the response from some view methods, which may be used in external integrations and off-chain tooling:

- `getStakingModule`;
- `getStakingModules`;
- `getStakingModuleDigests`;
- `getAllStakingModuleDigests`.

Tests show that [backward compatibility remains](https://github.com/lidofinance/sr-1.5-compatibility-tests) for both off-chain tools and possible on-chain integrations. The modified response is correctly decoded using standard Solidity tools and the Ethers library. New bytes in the response are ignored.

#### 2.2. Contract version upgrade

For correct migration to the new version of the `StakingRouter` contract, the existing `initialize` external method should be updated, and the new `finalizeUpgrade_v2` external method should be implemented. See more details about Lido proxy contracts upgrade requirements in the [LIP-10](https://github.com/lidofinance/lido-improvement-proposals/blob/feat/lip-24/LIPS/lip-10.md).

```solidity
/**
* @notice A function to finalize upgrade to v2 (from v1). Can be called only once
* @param _priorityExitShareThresholds array of priority exit share thresholds
* @param _maxDepositsPerBlock array of max deposits per block
* @param _minDepositBlockDistances array of min deposit block distances
*/
function finalizeUpgrade_v2(
    uint256[] memory _priorityExitShareThresholds,
    uint256[] memory _maxDepositsPerBlock,
    uint256[] memory _minDepositBlockDistances
) external {
    _checkContractVersion(1);

    uint256 stakingModulesCount = getStakingModulesCount();

    if (stakingModulesCount != _priorityExitShareThresholds.length) {
        revert ArraysLengthMismatch(stakingModulesCount, _priorityExitShareThresholds.length);
    }

    if (stakingModulesCount != _maxDepositsPerBlock.length) {
        revert ArraysLengthMismatch(stakingModulesCount, _maxDepositsPerBlock.length);
    }

    if (stakingModulesCount != _minDepositBlockDistances.length) {
        revert ArraysLengthMismatch(stakingModulesCount, _minDepositBlockDistances.length);
    }

    for (uint256 i; i < stakingModulesCount; ) {
        StakingModule storage stakingModule = _getStakingModuleByIndex(i);
        _updateStakingModule(
            stakingModule,
            stakingModule.id,
            stakingModule.stakeShareLimit,
            _priorityExitShareThresholds[i],
            stakingModule.stakingModuleFee,
            stakingModule.treasuryFee,
            _maxDepositsPerBlock[i],
            _minDepositBlockDistances[i]
        );

        unchecked {
            ++i;
        }
    }

    _updateContractVersion(2);
}
```

#### 2.3. Keys vetting improvements

Support of the [new key vetting logic](https://hackmd.io/@lido/rJrTnEc2a#Automated-Vetting) requires implementation of the new `decreaseStakingModuleVettedKeysCountByNodeOperator` external method. It is proposed to add a `STAKING_MODULE_UNVETTING_ROLE` and assign this role to the DSM.

```solidity
bytes32 public constant STAKING_MODULE_UNVETTING_ROLE = keccak256("STAKING_MODULE_UNVETTING_ROLE");

/// @notice decrese vetted signing keys counts per node operator for the staking module with
/// the specified id.
///
/// @param _stakingModuleId The id of the staking modules to be updated.
/// @param _nodeOperatorIds Ids of the node operators to be updated.
/// @param _vettedSigningKeysCounts New counts of vetted signing keys for the specified node operators.
///
function decreaseStakingModuleVettedKeysCountByNodeOperator(
    uint256 _stakingModuleId,
    bytes calldata _nodeOperatorIds,
    bytes calldata _vettedSigningKeysCounts
) external onlyRole(STAKING_MODULE_UNVETTING_ROLE) {
    _checkValidatorsByNodeOperatorReportData(_nodeOperatorIds, _vettedSigningKeysCounts);
    _getIStakingModuleById(_stakingModuleId).decreaseVettedSigningKeysCount(_nodeOperatorIds, _vettedSigningKeysCounts);
}
```

#### 2.4. New target limit modes

It is proposed to replace the current `isTargetLimitActive` parameter with the new `targetLimitMode` parameter to allow the Staking Router to consider the module's share when prioritizing validators for exit. More details can be found in the [`IStakingModule` interface specification](12-changes-in-existing-methods) and in the [VEBO Improvements specification](https://hackmd.io/@lido/BJXRTxMRp).

To implement this improvement the following changes are needed:
- The `isTargetLimitActive` field in the `NodeOperatorSummary` structure should be replaced with the new `targetLimitMode` field;
- Existing external `updateTargetValidatorsLimits` method should be updated;
- Existing `getNodeOperatorSummary` method should be updated.

```solidity
/// @notice A summary of node operator and its validators
struct NodeOperatorSummary {
    /// @notice Shows whether the current target limit applied to the node operator
    uint256 targetLimitMode;
    /// @notice Relative target active validators limit for operator
    uint256 targetValidatorsCount;
    /// @notice The number of validators with an expired request to exit time
    uint256 stuckValidatorsCount;
    /// @notice The number of validators that can't be withdrawn, but deposit costs were
    ///     compensated to the Lido by the node operator
    uint256 refundedValidatorsCount;
    /// @notice A time when the penalty for stuck validators stops applying to node operator rewards
    uint256 stuckPenaltyEndTimestamp;
    /// @notice The total number of validators in the EXITED state on the Consensus Layer
    /// @dev This value can't decrease in normal conditions
    uint256 totalExitedValidators;
    /// @notice The total number of validators deposited via the official Deposit Contract
    /// @dev This value is a cumulative counter: even when the validator goes into EXITED state this
    ///     counter is not decreasing
    uint256 totalDepositedValidators;
    /// @notice The number of validators in the set available for deposit
    uint256 depositableValidatorsCount;
}

/// @notice Updates the limit of the validators that can be used for deposit
/// @param _stakingModuleId Id of the staking module
/// @param _nodeOperatorId Id of the node operator
/// @param _targetLimitMode Target limit mode
/// @param _targetLimit Target limit of the node operator
function updateTargetValidatorsLimits(
    uint256 _stakingModuleId,
    uint256 _nodeOperatorId,
    uint256 _targetLimitMode,
    uint256 _targetLimit
) external onlyRole(STAKING_MODULE_MANAGE_ROLE) {
    _getIStakingModuleById(_stakingModuleId).updateTargetValidatorsLimits(
        _nodeOperatorId,
        _targetLimitMode,
        _targetLimit
    );
}

/// @notice Returns node operator summary from the staking module
/// @param _stakingModuleId id of the staking module where node operator is onboarded
/// @param _nodeOperatorId id of the node operator to return summary for
function getNodeOperatorSummary(
    uint256 _stakingModuleId,
    uint256 _nodeOperatorId
) public view returns (NodeOperatorSummary memory summary) {
    StakingModule memory stakingModuleState = getStakingModule(_stakingModuleId);
    IStakingModule stakingModule = IStakingModule(stakingModuleState.stakingModuleAddress);
    /// @dev using intermediate variables below due to "Stack too deep" error in case of
    ///     assigning directly into the NodeOperatorSummary struct
    (
        uint256 targetLimitMode,
        uint256 targetValidatorsCount,
        uint256 stuckValidatorsCount,
        uint256 refundedValidatorsCount,
        uint256 stuckPenaltyEndTimestamp,
        uint256 totalExitedValidators,
        uint256 totalDepositedValidators,
        uint256 depositableValidatorsCount
    ) = stakingModule.getNodeOperatorSummary(_nodeOperatorId);
    summary.targetLimitMode = targetLimitMode;
    summary.targetValidatorsCount = targetValidatorsCount;
    summary.stuckValidatorsCount = stuckValidatorsCount;
    summary.refundedValidatorsCount = refundedValidatorsCount;
    summary.stuckPenaltyEndTimestamp = stuckPenaltyEndTimestamp;
    summary.totalExitedValidators = totalExitedValidators;
    summary.totalDepositedValidators = totalDepositedValidators;
    summary.depositableValidatorsCount = depositableValidatorsCount;
}
```

#### 2.5. New module deposit limits

It is proposed to move the parameters `maxDepositsPerBlock` and `minDepositBlockDistance` from DSM to the Staking Router level. Modules with different properties have different risks when making deposits, so these parameters can be different for different modules. More reasons for this change are provided in the [DSM specification](https://hackmd.io/@lido/rJrTnEc2a#Deposit). New methods for getting these new parameters also should be implemented.

```solidity
function getStakingModuleMinDepositBlockDistance(uint256 _stakingModuleId) external view returns (uint256) {
    return _getStakingModuleById(_stakingModuleId).minDepositBlockDistance;
}

function getStakingModuleMaxDepositsPerBlock(uint256 _stakingModuleId) external view returns (uint256) {
    return _getStakingModuleById(_stakingModuleId).maxDepositsPerBlock;
}
```

##### 2.5.1. Backward compatibility notes

Changing the `StakingModule` struct will affect the following view methods, which can be used in external integrations and off-chain tooling:
- `getStakingModule`;
- `getStakingModules`;
- `getStakingModuleDigests`;
- `getAllStakingModuleDigests`.

Tests show that [backward compatibility remains](https://github.com/lidofinance/sr-1.5-compatibility-tests) for both off-chain tooling and possible on-chain integrations. The modified methods responses are correctly decoded by standard Solidity decoder and the Ethers library. New bytes in the responses are ignored.

#### 2.6. Excluded methods for module pausing
Now the protocol should be able to pause deposits to all modules at once. The pausing logic should be moved to the `DepositSecurityModule` contract. More reasons for this change are provided in the new [DSM specification](https://hackmd.io/@lido/rJrTnEc2a#Soft-Pause).

The following methods, events and constants that implement pausing logic in the `StakingRouter` contract should be deleted:

- `StakingModuleNotPaused` error type;
- `STAKING_MODULE_PAUSE_ROLE` and `STAKING_MODULE_RESUME_ROLE` constants;
- `pauseStakingModule` and `resumeStakingModule` methods.

### 3. `NodeOperatorsRegistry` contract changes

All the code of this contract assumes the Solidity v0.4.24 syntax.

#### 3.1. Contract initialization engine

The `initialize` method should now call the new internal `_initialize_v3` method. Also, the new `finalizeUpgrade_v3` external method should be implemented. See more details about Lido proxy contracts upgrade requirements in the [LIP-10](https://github.com/lidofinance/lido-improvement-proposals/blob/feat/lip-24/LIPS/lip-10.md).

```solidity
function initialize(address _locator, bytes32 _type, uint256 _stuckPenaltyDelay) public onlyInit {
    // Initializations for v1 --> v2
    _initialize_v2(_locator, _type, _stuckPenaltyDelay);

    // Initializations for v2 --> v3
    _initialize_v3();

    initialized();
}

function finalizeUpgrade_v3() external {
        require(hasInitialized(), "CONTRACT_NOT_INITIALIZED");
    _checkContractVersion(2);
    _initialize_v3();
}

function _initialize_v3() internal {
    _setContractVersion(3);
    _updateRewardDistributionState(RewardDistributionState.Distributed);
}
```

#### 3.2. New `decreaseVettedSigningKeysCount` method

The new `decreaseVettedSigningKeysCount` method should be implemented. It is called by the Staking Router to decrease the number of vetted keys for the node operator with the given ID. Also, the existing `setNodeOperatorStakingLimit` external method should be updated. See more details about this change in the [`IStakingModule` interface specification](#11-new-external-methods).

```solidity
/// @param _nodeOperatorIds bytes packed array of the node operators id
/// @param _vettedSigningKeysCounts bytes packed array of the new number of vetted keys for the node operators
function decreaseVettedSigningKeysCount(
    bytes _nodeOperatorIds,
    bytes _vettedSigningKeysCounts
) external {
    _auth(STAKING_ROUTER_ROLE);
    uint256 nodeOperatorsCount = _checkReportPayload(_nodeOperatorIds.length, _vettedSigningKeysCounts.length);
    uint256 totalNodeOperatorsCount = getNodeOperatorsCount();

    uint256 nodeOperatorId;
    uint256 vettedKeysCount;
    uint256 _nodeOperatorIdsOffset;
    uint256 _vettedKeysCountsOffset;

    /// @dev calldata layout:
    /// | func sig (4 bytes) | ABI-enc data |
    ///
    /// ABI-enc data:
    ///
    /// |    32 bytes    |     32 bytes      |  32 bytes  | ... |  32 bytes  | ...... |
    /// | ids len offset | counts len offset |  ids len   | ids | counts len | counts |
    assembly {
        _nodeOperatorIdsOffset := add(calldataload(4), 36) // arg1 calldata offset + 4 (signature len) + 32 (length slot)
        _vettedKeysCountsOffset := add(calldataload(36), 36) // arg2 calldata offset + 4 (signature len) + 32 (length slot))
    }
    for (uint256 i; i < nodeOperatorsCount;) {
        /// @solidity memory-safe-assembly
        assembly {
            nodeOperatorId := shr(192, calldataload(add(_nodeOperatorIdsOffset, mul(i, 8))))
            vettedKeysCount := shr(128, calldataload(add(_vettedKeysCountsOffset, mul(i, 16))))
            i := add(i, 1)
        }
        _requireValidRange(nodeOperatorId < totalNodeOperatorsCount);
        _updateVettedSingingKeysCount(nodeOperatorId, vettedKeysCount, false /* only decrease */);
    }
    _increaseValidatorsKeysNonce();
}

function setNodeOperatorStakingLimit(uint256 _nodeOperatorId, uint64 _vettedSigningKeysCount) external {
    _onlyExistedNodeOperator(_nodeOperatorId);
    _authP(SET_NODE_OPERATOR_LIMIT_ROLE, arr(uint256(_nodeOperatorId), uint256(_vettedSigningKeysCount)));
    _onlyCorrectNodeOperatorState(getNodeOperatorIsActive(_nodeOperatorId));

    _updateVettedSingingKeysCount(_nodeOperatorId, _vettedSigningKeysCount, true /* _allowIncrease */);
    _increaseValidatorsKeysNonce();
}
```

#### 3.3. New reward distribution engine

Currently curated modules distribute rewards during the third phase of the Accounting Oracle report, while other modules can use alternative mechanisms of rewards distribution. This complicates the accounting report and could potentially become a source of bugs.

To solve this problem it is proposed to implement new permissionless `distributeReward` method. This method should distribute all accumulated module rewards among node operators based on the latest accounting report. Using this new method, reward distribution will be decoupled from the Accounting Oracle report and can be executed separately in each staking module.

Rewards can be distributed after node operators' statistics are updated until the next reward is transferred to the module during the next oracle frame.

---

**Start report frame 1**
1. Oracle first phase: Reach hash consensus.
2. Oracle second phase: Module receives rewards.
3. Oracle third phase: Operator statistics are updated.

*... Reward can be distributed ...*

**Start report frame 2**

*... Reward can be distributed ...*
*(if not distributed yet)*

1. Oracle first phase: Reach hash consensus.
2. Oracle second phase: Module receives rewards.

*... Reward CANNOT be distributed ...*

3. Oracle third phase: Operator statistics are updated.

*... Reward can be distributed ...*

**Start report frame 3**

---

Once the delivery of third-phase report updates is complete, the Accounting Oracle triggers the Finalization hook (calling the Staking Router's `onValidatorsCountsByNodeOperatorReportingFinished`). Within this method, for each staking module, the `onExitedAndStuckValidatorsCountsUpdated` method should be called. This signals that all Node Operator updates have been successfully delivered and the Accounting Oracle report is finalized.

Currently, rewards distribution in curated-based staking modules occurs within the `onExitedAndStuckValidatorsCountsUpdated` method. The proposed solution is to update this method to mark the module as ready for reward distribution instead of distributing the reward directly. The actual reward distribution will subsequently start within the `distributeReward` method.

It's worth mentioning the existing `onRewardsMinted` method, which is used by the Staking Router during the second phase of the Accounting Oracle report to notify each Staking Module that the total module reward has been transferred to the module. In this method, reward distribution is blocked until the third phase report is finished (as per `onExitedAndStuckValidatorsCountsUpdated` above), ensuring that rewards can be distributed among node operators.

More details about the new reward distribution engine can be found in the [Reward distribution in curated-based modules specification](https://hackmd.io/@lido/HJYbVq5b0).

```solidity
event RewardDistributionStateChanged(RewardDistributionState state);

// Enum to represent the state of the reward distribution process
enum RewardDistributionState {
    TransferredToModule,      // New reward portion minted and transferred to the module
    ReadyForDistribution,     // Operators' statistics updated, reward ready for distribution
    Distributed               // Reward distributed among operators
}

// bytes32 internal constant REWARD_DISTRIBUTION_STATE = keccak256("lido.NodeOperatorsRegistry.rewardDistributionState");
bytes32 internal constant REWARD_DISTRIBUTION_STATE = 0x4ddbb0dcdc5f7692e494c15a7fca1f9eb65f31da0b5ce1c3381f6a1a1fd579b6;

function distributeReward() external {
    require(getRewardDistributionState() == RewardDistributionState.ReadyForDistribution, "DISTRIBUTION_NOT_READY");
    _updateRewardDistributionState(RewardDistributionState.Distributed);
    _distributeRewards();
}

/// @dev Get the current reward distribution state, anyone can monitor this state
/// and distribute reward (call distributeReward method) among operators when it's `ReadyForDistribution`
function getRewardDistributionState() public view returns (RewardDistributionState) {
    uint256 state = REWARD_DISTRIBUTION_STATE.getStorageUint256();
    return RewardDistributionState(state);
}

function _updateRewardDistributionState(RewardDistributionState _state) internal {
    REWARD_DISTRIBUTION_STATE.setStorageUint256(uint256(_state));
    emit RewardDistributionStateChanged(_state);
}

function onRewardsMinted(uint256 /* _totalShares */) external {
    _auth(STAKING_ROUTER_ROLE);
    _updateRewardDistributionState(RewardDistributionState.TransferredToModule);
}

function onExitedAndStuckValidatorsCountsUpdated() external {
    _auth(STAKING_ROUTER_ROLE);
    _updateRewardDistributionState(RewardDistributionState.ReadyForDistribution);
}
```

#### 3.4. New target limit modes

Now it should be possible to have 3 possible states for the target validators limit. More details about the purpose of this change can be found in the [`IStakingModule` interface specification](#12-changes-in-existing-methods) and in the [VEBO Improvements specification](https://hackmd.io/@lido/BJXRTxMRp).

The `updateTargetValidatorsLimits` public method should be updated to support this change. The existing `TargetValidatorsCountChanged` event also should be updated.

```solidity
event TargetValidatorsCountChanged(uint256 indexed nodeOperatorId, uint256 targetValidatorsCount, uint256 targetLimitMode);

/// Target limit mode, allows limiting target active validators count for operator (0 = disabled, 1 = soft mode, 2 = forced mode)
uint8 internal constant TARGET_LIMIT_MODE_OFFSET = 0;

function updateTargetValidatorsLimits(uint256 _nodeOperatorId, uint256 _targetLimitMode, uint256 _targetLimit) public {
    _onlyExistedNodeOperator(_nodeOperatorId);
    _auth(STAKING_ROUTER_ROLE);
    _requireValidRange(_targetLimit <= UINT64_MAX);

    Packed64x4.Packed memory operatorTargetStats = _loadOperatorTargetValidatorsStats(_nodeOperatorId);
    operatorTargetStats.set(TARGET_LIMIT_MODE_OFFSET, _targetLimitMode);
    if (_targetLimitMode == 0) {
        _targetLimit = 0;
    }
    operatorTargetStats.set(TARGET_VALIDATORS_COUNT_OFFSET, _targetLimit);
    _saveOperatorTargetValidatorsStats(_nodeOperatorId, operatorTargetStats);

    emit TargetValidatorsCountChanged(_nodeOperatorId, _targetLimit, _targetLimitMode);

    _updateSummaryMaxValidatorsCount(_nodeOperatorId);
    _increaseValidatorsKeysNonce();
}

function getNodeOperatorSummary(uint256 _nodeOperatorId)
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
) {
    _onlyExistedNodeOperator(_nodeOperatorId);

    Packed64x4.Packed memory operatorTargetStats = _loadOperatorTargetValidatorsStats(_nodeOperatorId);
    Packed64x4.Packed memory stuckPenaltyStats = _loadOperatorStuckPenaltyStats(_nodeOperatorId);

    targetLimitMode = operatorTargetStats.get(TARGET_LIMIT_MODE_OFFSET);
    targetValidatorsCount = operatorTargetStats.get(TARGET_VALIDATORS_COUNT_OFFSET);
    stuckValidatorsCount = stuckPenaltyStats.get(STUCK_VALIDATORS_COUNT_OFFSET);
    refundedValidatorsCount = stuckPenaltyStats.get(REFUNDED_VALIDATORS_COUNT_OFFSET);
    stuckPenaltyEndTimestamp = stuckPenaltyStats.get(STUCK_PENALTY_END_TIMESTAMP_OFFSET);

    (totalExitedValidators, totalDepositedValidators, depositableValidatorsCount) =
        _getNodeOperatorValidatorsSummary(_nodeOperatorId);
}
```

### 4. `DepositSecurityModule` contract changes

All the code of this contract assumes the Solidity v0.8.9 syntax.

#### 4.1. Changes in `IStakingRouter` interface

The `IStakingRouter` interface should be changed to reflect the new Deposit Security Module logic. The `pauseStakingModule`, `resumeStakingModule` and `getStakingModuleIsDepositsPaused` functions should be removed from the interface, and new `getStakingModuleMinDepositBlockDistance`, `getStakingModuleMaxDepositsPerBlock` and `decreaseStakingModuleVettedKeysCountByNodeOperator` functions should be added.

More details about these changes can be found in the new [DSM specification](https://hackmd.io/@lido/rJrTnEc2a).

```solidity
interface IStakingRouter {
    function getStakingModuleMinDepositBlockDistance(uint256 _stakingModuleId) external view returns (uint256);
    function getStakingModuleMaxDepositsPerBlock(uint256 _stakingModuleId) external view returns (uint256);
    function getStakingModuleIsActive(uint256 _stakingModuleId) external view returns (bool);
    function getStakingModuleNonce(uint256 _stakingModuleId) external view returns (uint256);
    function getStakingModuleLastDepositBlock(uint256 _stakingModuleId) external view returns (uint256);
    function hasStakingModule(uint256 _stakingModuleId) external view returns (bool);
    function decreaseStakingModuleVettedKeysCountByNodeOperator(
        uint256 _stakingModuleId,
        bytes calldata _nodeOperatorIds,
        bytes calldata _vettedSigningKeysCounts
    ) external;
}
```

#### 4.2. New keys vetting logic

Keys may become invalid over time in case of a front-run attack attempt. Therefore, it's important to [introduce a mechanism for unvetting keys](https://hackmd.io/@lido/rJrTnEc2a#Unvetting), which will be uniform and required for all modules. It is proposed to grant the role of decreasing the number of vetted keys to DSM and to modify the `IStakingModule` interface that must be supported by every module. Thus, unvetting occurs through the DSM contract, which calls the corresponding method on the module side through the Staking Router.

This requires a bunch of new events and methods to be defined:
- New `unvetSigningKeys` external method that unvets signing keys for the given node operators. This method is supposed to be called by the Council Daemon if it finds an invalid key in the deposit queue;
- Two new `getMaxOperatorsPerUnvetting` and `setMaxOperatorsPerUnvetting` external methods that get and set the maximum number of operators per unvetting;
- New `_setMaxOperatorsPerUnvetting` internal method that emits the `MaxOperatorsPerUnvettingChanged` event;
- New `MaxOperatorsPerUnvettingChanged` event;
- Two new `UnvetPayloadInvalid` and `UnvetUnexpectedBlockHash` error types;
- New `maxOperatorsPerUnvetting` field;
- New `UNVET_MESSAGE_PREFIX` prefix for the message signed by guardians to unvet signing keys.

Existing `MaxDepositsChanged` and `MinDepositBlockDistanceChanged` events, `getMaxDeposits`, `setMaxDeposits`, `getMinDepositBlockDistance` and `setMinDepositBlockDistance` external methods, `_setMaxDeposits` and `_setMinDepositBlockDistance` internal methods should be removed from the contract implementation.

```solidity
event MaxOperatorsPerUnvettingChanged(uint256 newValue);

error UnvetPayloadInvalid();
error UnvetUnexpectedBlockHash();

bytes32 public immutable UNVET_MESSAGE_PREFIX;

uint256 internal maxOperatorsPerUnvetting;

/**
* @return maxOperatorsPerUnvetting The maximum number of operators per unvetting.
*/
function getMaxOperatorsPerUnvetting() external view returns (uint256) {
    return maxOperatorsPerUnvetting;
}

/**
* @param newValue The new maximum number of operators per unvetting.
* @dev Only callable by the owner.
*/
function setMaxOperatorsPerUnvetting(uint256 newValue) external onlyOwner {
    _setMaxOperatorsPerUnvetting(newValue);
}

function _setMaxOperatorsPerUnvetting(uint256 newValue) internal {
    if (newValue == 0) revert ZeroParameter("maxOperatorsPerUnvetting");
    maxOperatorsPerUnvetting = newValue;
    emit MaxOperatorsPerUnvettingChanged(newValue);
}

/**
* @param blockNumber The block number at which the unvet intent was created.
* @param blockHash The block hash at which the unvet intent was created.
* @param stakingModuleId The ID of the staking module.
* @param nonce The nonce of the staking module.
* @param nodeOperatorIds The list of node operator IDs.
* @param vettedSigningKeysCounts The list of vetted signing keys counts.
* @param sig The signature of the guardian.
* @dev Reverts if any of the following is true:
*   - The nonce is not equal to the on-chain nonce of the staking module;
*   - nodeOperatorIds is not packed with 8 bytes per id;
*   - vettedSigningKeysCounts is not packed with 16 bytes per count;
*   - the number of node operators is greater than maxOperatorsPerUnvetting;
*   - the signature is invalid or the signer is not a guardian;
*   - blockHash is zero or not equal to the blockhash(blockNumber).
*
* The signature, if present, must be produced for the keccak256 hash of the following message:
*
* | UNVET_MESSAGE_PREFIX | blockNumber | blockHash | stakingModuleId | nonce | nodeOperatorIds | vettedSigningKeysCounts |
*/
function unvetSigningKeys(
    uint256 blockNumber,
    bytes32 blockHash,
    uint256 stakingModuleId,
    uint256 nonce,
    bytes calldata nodeOperatorIds,
    bytes calldata vettedSigningKeysCounts,
    Signature calldata sig
) external {
    /// @dev The most likely reason for the signature to go stale
    uint256 onchainNonce = STAKING_ROUTER.getStakingModuleNonce(stakingModuleId);
    if (nonce != onchainNonce) revert ModuleNonceChanged();

    uint256 nodeOperatorsCount = nodeOperatorIds.length / 8;

    if (
        nodeOperatorIds.length % 8 != 0 ||
        vettedSigningKeysCounts.length % 16 != 0 ||
        vettedSigningKeysCounts.length / 16 != nodeOperatorsCount ||
        nodeOperatorsCount > maxOperatorsPerUnvetting
    ) {
        revert UnvetPayloadInvalid();
    }

    address guardianAddr = msg.sender;
    int256 guardianIndex = _getGuardianIndex(msg.sender);

    if (guardianIndex == -1) {
        bytes32 msgHash = keccak256(
            // slither-disable-start encode-packed-collision
            // values with a dynamic type checked earlier
            abi.encodePacked(
                UNVET_MESSAGE_PREFIX,
                blockNumber,
                blockHash,
                stakingModuleId,
                nonce,
                nodeOperatorIds,
                vettedSigningKeysCounts
            )
            // slither-disable-end encode-packed-collision
        );
        guardianAddr = ECDSA.recover(msgHash, sig.r, sig.vs);
        guardianIndex = _getGuardianIndex(guardianAddr);
        if (guardianIndex == -1) revert InvalidSignature();
    }

    if (blockHash == bytes32(0) || blockhash(blockNumber) != blockHash) revert UnvetUnexpectedBlockHash();

    STAKING_ROUTER.decreaseStakingModuleVettedKeysCountByNodeOperator(
        stakingModuleId,
        nodeOperatorIds,
        vettedSigningKeysCounts
    );
}
```

#### 4.3. New deposits pause logic

To support new permissionless modules and eliminate guardians collusion attack, now it should be possible to pause deposits to all modules at once. [The proposed design](https://hackmd.io/@lido/rJrTnEc2a#Pause) no longer allows an operator to trigger a pause, which eliminates the need to isolate pauses by modules and implements an approach of a universal deposit pause, reducing the risks of a guardian collusion attack.

New deposits pause logic requires the following changes:
- Existing `pauseDeposits` and `unpauseDeposits` external methods should be changed;
- Signature of the existing `DepositsPaused` and `DepositsUnpaused` events should be changed;
- Two new `DepositsArePaused` and `DepositsNotPaused` error types should be created;
- New public flag indicating whether deposits are paused should be created.

```solidity
event DepositsPaused(address indexed guardian);
event DepositsUnpaused();

error DepositsArePaused();
error DepositsNotPaused();

bool public isDepositsPaused;

/**
* @param blockNumber The block number at which the pause intent was created.
* @param sig The signature of the guardian.
* @dev Does nothing if deposits are already paused.
*
* Reverts if:
*  - the pause intent is expired;
*  - the caller is not a guardian and signature is invalid or not from a guardian.
*
* The signature, if present, must be produced for the keccak256 hash of the following
* message (each component taking 32 bytes):
*
* | PAUSE_MESSAGE_PREFIX | blockNumber |
*/
function pauseDeposits(uint256 blockNumber, Signature memory sig) external {
    /// @dev In case of an emergency function `pauseDeposits` is supposed to be called
    /// by all guardians. Thus only the first call will do the actual change. But
    /// the other calls would be OK operations from the point of view of protocol’s logic.
    /// Thus we prefer not to use “error” semantics which is implied by `require`.
    if (isDepositsPaused) return;

    address guardianAddr = msg.sender;
    int256 guardianIndex = _getGuardianIndex(msg.sender);

    if (guardianIndex == -1) {
        bytes32 msgHash = keccak256(abi.encodePacked(PAUSE_MESSAGE_PREFIX, blockNumber));
        guardianAddr = ECDSA.recover(msgHash, sig.r, sig.vs);
        guardianIndex = _getGuardianIndex(guardianAddr);
        if (guardianIndex == -1) revert InvalidSignature();
    }

    if (block.number - blockNumber > pauseIntentValidityPeriodBlocks) revert PauseIntentExpired();

    isDepositsPaused = true;
    emit DepositsPaused(guardianAddr);
}

/**
* @dev Only callable by the owner.
* Reverts if deposits are not paused.
*/
function unpauseDeposits() external onlyOwner {
    if (!isDepositsPaused) revert DepositsNotPaused();
    isDepositsPaused = false;
    emit DepositsUnpaused();
}
```

#### 4.4. Deposits frequency limits

The frequency of deposits now should be limited. This way there will be distance between deposits to different modules, similar to deposits within a single module. Deposit frequency checks in `depositBufferedEther` and `canDeposit` methods should use the maximum value of `lastDepositBlock` from the module and DSM contract.

This improvement requires a number of changes:
- Existing `canDeposit` external method should be updated;
- Existing `depositBufferedEther` external method should be updated;
- New `isMinDepositDistancePassed` external method should be implemented;
- New `lastDepositBlock` field should be added;
- New `LastDepositBlockChanged` event should be added;
- New `ModuleNonceChanged` error type should be added.

```solidity
event LastDepositBlockChanged(uint256 newValue);

error ModuleNonceChanged();

uint256 internal lastDepositBlock;

/**
* @notice Returns whether LIDO.deposit() can be called, given that the caller
* will provide guardian attestations of non-stale deposit root and nonce,
* and the number of such attestations will be enough to reach the quorum.
*
* @param stakingModuleId The ID of the staking module.
* @return canDeposit Whether a deposit can be made.
* @dev Returns true if all of the following conditions are met:
*   - deposits are not paused;
*   - the staking module is active;
*   - the guardian quorum is not set to zero;
*   - the deposit distance is greater than the minimum required;
*   - LIDO.canDeposit() returns true.
*/
function canDeposit(uint256 stakingModuleId) external view returns (bool) {
    if (!STAKING_ROUTER.hasStakingModule(stakingModuleId)) return false;

    bool isModuleActive = STAKING_ROUTER.getStakingModuleIsActive(stakingModuleId);
    bool isDepositDistancePassed = _isMinDepositDistancePassed(stakingModuleId);
    bool isLidoCanDeposit = LIDO.canDeposit();

    return (
        !isDepositsPaused
        && isModuleActive
        && quorum > 0
        && isDepositDistancePassed
        && isLidoCanDeposit
    );
}

/**
* @notice Calls LIDO.deposit(maxDepositsPerBlock, stakingModuleId, depositCalldata).
* @param blockNumber The block number at which the deposit intent was created.
* @param blockHash The block hash at which the deposit intent was created.
* @param depositRoot The deposit root hash.
* @param stakingModuleId The ID of the staking module.
* @param nonce The nonce of the staking module.
* @param depositCalldata The calldata for the deposit.
* @param sortedGuardianSignatures The list of guardian signatures ascendingly sorted by address.
* @dev Reverts if any of the following is true:
*   - quorum is zero;
*   - the number of guardian signatures is less than the quorum;
*   - onchain deposit root is different from the provided one;
*   - module is not active;
*   - min deposit distance is not passed;
*   - blockHash is zero or not equal to the blockhash(blockNumber);
*   - onchain module nonce is different from the provided one;
*   - invalid or non-guardian signature received;
*
* Signatures must be sorted in ascending order by address of the guardian. Each signature must
* be produced for the keccak256 hash of the following message (each component taking 32 bytes):
*
* | ATTEST_MESSAGE_PREFIX | blockNumber | blockHash | depositRoot | stakingModuleId | nonce |
*/
function depositBufferedEther(
    uint256 blockNumber,
    bytes32 blockHash,
    bytes32 depositRoot,
    uint256 stakingModuleId,
    uint256 nonce,
    bytes calldata depositCalldata,
    Signature[] calldata sortedGuardianSignatures
) external {
    /// @dev The first most likely reason for the signature to go stale
    bytes32 onchainDepositRoot = IDepositContract(DEPOSIT_CONTRACT).get_deposit_root();
    if (depositRoot != onchainDepositRoot) revert DepositRootChanged();

    /// @dev The second most likely reason for the signature to go stale
    uint256 onchainNonce = STAKING_ROUTER.getStakingModuleNonce(stakingModuleId);
    if (nonce != onchainNonce) revert ModuleNonceChanged();

    if (quorum == 0 || sortedGuardianSignatures.length < quorum) revert DepositNoQuorum();
    if (!STAKING_ROUTER.getStakingModuleIsActive(stakingModuleId)) revert DepositInactiveModule();
    if (!_isMinDepositDistancePassed(stakingModuleId)) revert DepositTooFrequent();
    if (blockHash == bytes32(0) || blockhash(blockNumber) != blockHash) revert DepositUnexpectedBlockHash();
    if (isDepositsPaused) revert DepositsArePaused();

    _verifyAttestSignatures(depositRoot, blockNumber, blockHash, stakingModuleId, nonce, sortedGuardianSignatures);

    uint256 maxDepositsPerBlock = STAKING_ROUTER.getStakingModuleMaxDepositsPerBlock(stakingModuleId);
    LIDO.deposit(maxDepositsPerBlock, stakingModuleId, depositCalldata);

    _setLastDepositBlock(block.number);
}

/**
* @notice Returns whether the deposit distance is greater than the minimum required.
* @param stakingModuleId The ID of the staking module.
* @return isMinDepositDistancePassed Whether the deposit distance is greater than the minimum required.
* @dev Checks the distance for the last deposit to any staking module.
*/
function isMinDepositDistancePassed(uint256 stakingModuleId) external view returns (bool) {
    return _isMinDepositDistancePassed(stakingModuleId);
}

function _isMinDepositDistancePassed(uint256 stakingModuleId) internal view returns (bool) {
    uint256 lastDepositToModuleBlock = STAKING_ROUTER.getStakingModuleLastDepositBlock(stakingModuleId);
    uint256 minDepositBlockDistance = STAKING_ROUTER.getStakingModuleMinDepositBlockDistance(stakingModuleId);
    uint256 maxLastDepositBlock = lastDepositToModuleBlock >= lastDepositBlock ? lastDepositToModuleBlock : lastDepositBlock;
    return block.number - maxLastDepositBlock >= minDepositBlockDistance;
}
```

#### 4.5. Changes in contract constructor

Suggested improvements require changes in the contract constructor.

```solidity
/**
* @notice Initializes the contract with the given parameters.
* @dev Reverts if any of the addresses is zero.
*
* Sets the last deposit block to the current block number.
*/
constructor(
    address _lido,
    address _depositContract,
    address _stakingRouter,
    uint256 _pauseIntentValidityPeriodBlocks,
    uint256 _maxOperatorsPerUnvetting
) {
    if (_lido == address(0)) revert ZeroAddress("_lido");
    if (_depositContract == address(0)) revert ZeroAddress("_depositContract");
    if (_stakingRouter == address(0)) revert ZeroAddress("_stakingRouter");

    LIDO = ILido(_lido);
    STAKING_ROUTER = IStakingRouter(_stakingRouter);
    DEPOSIT_CONTRACT = IDepositContract(_depositContract);

    ATTEST_MESSAGE_PREFIX = keccak256(
        abi.encodePacked(
            // keccak256("lido.DepositSecurityModule.ATTEST_MESSAGE")
            bytes32(0x1085395a994e25b1b3d0ea7937b7395495fb405b31c7d22dbc3976a6bd01f2bf),
            block.chainid,
            address(this)
        )
    );

    PAUSE_MESSAGE_PREFIX = keccak256(
        abi.encodePacked(
            // keccak256("lido.DepositSecurityModule.PAUSE_MESSAGE")
            bytes32(0x9c4c40205558f12027f21204d6218b8006985b7a6359bcab15404bcc3e3fa122),
            block.chainid,
            address(this)
        )
    );

    UNVET_MESSAGE_PREFIX = keccak256(
        abi.encodePacked(
            // keccak256("lido.DepositSecurityModule.UNVET_MESSAGE")
            bytes32(0x2dd9727393562ed11c29080a884630e2d3a7078e71b313e713a8a1ef68948f6a),
            block.chainid,
            address(this)
        )
    );

    _setOwner(msg.sender);
    _setLastDepositBlock(block.number);
    _setPauseIntentValidityPeriodBlocks(_pauseIntentValidityPeriodBlocks);
    _setMaxOperatorsPerUnvetting(_maxOperatorsPerUnvetting);
}
```

#### 4.6. New code version constant

New code version constants that helps to distinguish contract interfaces should be added to the contract.

```solidity
uint256 public constant VERSION = 3;
```

### 5. `OracleReportSanityChecker` contract changes

With the introduction of new modules, the Accounting Oracle third-phase report might not fit into a single transaction. In the new version of the Accounting Oracle, the ability to split the report's third phase to multiple transactions will be implemented. To support these changes, some of the on-chain sanity checkers in the `OracleReportSanityChecker` contract should be reconsidered.

Currently, the `churnValidatorsPerDayLimit` parameter is used to validate both newly exited validators `checkExitedValidatorsRatePerDay` and newly deposited validators `_checkAppearedValidatorsChurnLimit`. It is proposed to divide this into two distinct parameters, each with more precise values tailored to the respective cases. On the second phase of the Accounting Report, the `checkExitedValidatorsRatePerDay` sanity check will use new `exitedValidatorsPerDayChurnLimit`, and the `_checkAppearedValidatorsChurnLimit` check will use the value from the `appearedValidatorsPerDayLimit`.

- New `exitedValidatorsPerDayLimit` parameter is the maximum possible number of validators that might be reported as `exited` per single day, depends on the Consensus Layer churn limit.
- New `appearedValidatorsPerDayLimit` parameter is the maximum possible number of validators that might be reported as `appeared` per single day.

All related structures, constants and methods of the `OracleReportSanityChecker` contract should be updated to support these two new limits.

More details about new sanity checkers can be found in the [Expand the third phase in Oracle specification](https://hackmd.io/HCXVrrZAQNCm_IC0lYvJjw).

All the code of this contract assumes the Solidity v0.8.9 syntax.

```solidity
struct LimitsList {
    /// ...
    
    /// @dev Must fit into uint16 (<= 65_535)
    uint256 exitedValidatorsPerDayLimit;

    /// @dev Must fit into uint16 (<= 65_535)
    uint256 appearedValidatorsPerDayLimit;
    
    /// ...
}

bytes32 public constant EXITED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE =
        keccak256("EXITED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE");

bytes32 public constant APPEARED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE =
        keccak256("APPEARED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE");

/// @notice Sets the new value for the exitedValidatorsPerDayLimit
///
/// NB: AccountingOracle reports validators as exited once they passed the `EXIT_EPOCH` on Consensus Layer
///     therefore, the value should be set in accordance to the consensus layer churn limit
///
/// @param _exitedValidatorsPerDayLimit new exitedValidatorsPerDayLimit value
function setExitedValidatorsPerDayLimit(uint256 _exitedValidatorsPerDayLimit)
    external
    onlyRole(EXITED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE)
{
    LimitsList memory limitsList = _limits.unpack();
    limitsList.exitedValidatorsPerDayLimit = _exitedValidatorsPerDayLimit;
    _updateLimits(limitsList);
}

/// @notice Sets the new value for the appearedValidatorsPerDayLimit
///
/// NB: AccountingOracle reports validators as appeared once they become `pending`
///     (might be not `activated` yet). Thus, this limit should be high enough because consensus layer
///     has no intrinsic churn limit for the amount of `pending` validators (only for `activated` instead).
///     For Lido it depends on the amount of deposits that can be made via DepositSecurityModule daily.
///
/// @param _appearedValidatorsPerDayLimit new appearedValidatorsPerDayLimit value
function setAppearedValidatorsPerDayLimit(uint256 _appearedValidatorsPerDayLimit)
    external
    onlyRole(APPEARED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE)
{
    LimitsList memory limitsList = _limits.unpack();
    limitsList.appearedValidatorsPerDayLimit = _appearedValidatorsPerDayLimit;
    _updateLimits(limitsList);
}

/// @notice Check rate of exited validators per day
/// @param _exitedValidatorsCount Number of validator exit requests supplied per oracle report
function checkExitedValidatorsRatePerDay(uint256 _exitedValidatorsCount)
    external
    view
{
    uint256 exitedValidatorsLimit = _limits.unpack().exitedValidatorsPerDayLimit;
    if (_exitedValidatorsCount > exitedValidatorsLimit) {
        revert ExitedValidatorsLimitExceeded(exitedValidatorsLimit, _exitedValidatorsCount);
    }
}

event ExitedValidatorsPerDayLimitSet(uint256 exitedValidatorsPerDayLimit);
event AppearedValidatorsPerDayLimitSet(uint256 appearedValidatorsPerDayLimit);
```

### 6. `AccountingOracle` contract changes

All the code of this contract assumes the Solidity v0.8.9 syntax.

#### 6.1. Multi-transactional third phase

To support multi-transactional third phase of the Accounting Oracle, a number of changes should be implemented in the `AccountingOracle` contract:
- The `IOracleReportSanityChecker` call should be removed from the existing `_handleConsensusReportData` internal method;
- Implementation of the `_submitReportExtraDataList` and `_processExtraDataItems` internal methods should be changed;
- Second and third phases of the `_processExtraDataItem` internal methods should be updated;
- Existing `ExtraDataListOnlySupportsSingleTx` error type should be replaced with the new `UnexpectedExtraDataLength` error type.

More implementation details can be found in the [Expand the third phase in Oracle specification](https://hackmd.io/HCXVrrZAQNCm_IC0lYvJjw?view#Implementation-Details).

```solidity
error UnexpectedExtraDataLength();

bytes32 internal constant ZERO_HASH = bytes32(0);

function _submitReportExtraDataList(bytes calldata data) internal {
    ExtraDataProcessingState memory procState = _storageExtraDataProcessingState().value;
    _checkCanSubmitExtraData(procState, EXTRA_DATA_FORMAT_LIST);

    if (procState.itemsProcessed == procState.itemsCount) {
        revert ExtraDataAlreadyProcessed();
    }

    // at least 32 bytes for the next hash value + 35 bytes for the first item with 1 node operator
    if(data.length < 67) {
        revert UnexpectedExtraDataLength();
    }

    bytes32 dataHash = keccak256(data);
    if (dataHash != procState.dataHash) {
        revert UnexpectedExtraDataHash(procState.dataHash, dataHash);
    }

    // load the next hash value
    assembly {
        dataHash := calldataload(data.offset)
    }

    ExtraDataIterState memory iter = ExtraDataIterState({
        index: procState.itemsProcessed > 0 ? procState.itemsProcessed - 1 : 0,
        itemType: 0,
        dataOffset: 32, // skip the next hash bytes
        lastSortingKey: procState.lastSortingKey,
        stakingRouter: LOCATOR.stakingRouter()
    });

    _processExtraDataItems(data, iter);
    uint256 itemsProcessed = iter.index + 1;

    if(dataHash == ZERO_HASH) {
        if (itemsProcessed != procState.itemsCount) {
            revert UnexpectedExtraDataItemsCount(procState.itemsCount, itemsProcessed);
        }

        procState.submitted = true;
        procState.itemsProcessed = uint64(itemsProcessed);
        procState.lastSortingKey = iter.lastSortingKey;
        _storageExtraDataProcessingState().value = procState;

        IStakingRouter(iter.stakingRouter).onValidatorsCountsByNodeOperatorReportingFinished();
    } else {
        if (itemsProcessed >= procState.itemsCount) {
            revert UnexpectedExtraDataItemsCount(procState.itemsCount, itemsProcessed);
        }

        // save the next hash value
        procState.dataHash = dataHash;
        procState.itemsProcessed = uint64(itemsProcessed);
        procState.lastSortingKey = iter.lastSortingKey;
        _storageExtraDataProcessingState().value = procState;
    }

    emit ExtraDataSubmitted(procState.refSlot, procState.itemsProcessed, procState.itemsCount);
}

function _processExtraDataItems(bytes calldata data, ExtraDataIterState memory iter) internal {
    uint256 dataOffset = iter.dataOffset;
    uint256 maxNodeOperatorsPerItem = 0;
    uint256 maxNodeOperatorItemIndex = 0;
    uint256 itemsCount;
    uint256 index;
    uint256 itemType;

    while (dataOffset < data.length) {
        /// @solidity memory-safe-assembly
        assembly {
            // layout at the dataOffset:
            // |  3 bytes  | 2 bytes  |   X bytes   |
            // | itemIndex | itemType | itemPayload |
            let header := calldataload(add(data.offset, dataOffset))
            index := shr(232, header)
            itemType := and(shr(216, header), 0xffff)
            dataOffset := add(dataOffset, 5)
        }

        if (iter.lastSortingKey == 0) {
            if (index != 0) {
                revert UnexpectedExtraDataIndex(0, index);
            }
        } else if (index != iter.index + 1) {
            revert UnexpectedExtraDataIndex(iter.index + 1, index);
        }

        iter.index = index;
        iter.itemType = itemType;
        iter.dataOffset = dataOffset;

        if (itemType == EXTRA_DATA_TYPE_EXITED_VALIDATORS ||
            itemType == EXTRA_DATA_TYPE_STUCK_VALIDATORS
        ) {
            uint256 nodeOpsProcessed = _processExtraDataItem(data, iter);

            if (nodeOpsProcessed > maxNodeOperatorsPerItem) {
                maxNodeOperatorsPerItem = nodeOpsProcessed;
                maxNodeOperatorItemIndex = index;
            }
        } else {
            revert UnsupportedExtraDataType(index, itemType);
        }

        assert(iter.dataOffset > dataOffset);
        dataOffset = iter.dataOffset;
        unchecked {
            // overflow is not possible here
            ++itemsCount;
        }
    }

    assert(maxNodeOperatorsPerItem > 0);

    IOracleReportSanityChecker(LOCATOR.oracleReportSanityChecker())
        .checkExtraDataItemsCountPerTransaction(itemsCount);

    IOracleReportSanityChecker(LOCATOR.oracleReportSanityChecker())
        .checkNodeOperatorsPerExtraDataItemCount(maxNodeOperatorItemIndex, maxNodeOperatorsPerItem);
}
```

#### 6.2. Sanity checkers

Currently the `OracleReportSanityChecker` contains parameters to establish the maximum number of node operators that can be updated during the third phase of the report. Sanity checks in the `IOracleReportSanityChecker` interface should be changed as the multi-transaction approach no longer imposes limits on the entire report size. Rather than validating the entire report size, it is proposed to validate each transaction during the third phase.

```solidity
interface IOracleReportSanityChecker {
    function checkExitedValidatorsRatePerDay(uint256 _exitedValidatorsCount) external view;
    function checkExtraDataItemsCountPerTransaction(uint256 _extraDataListItemsCount) external view;
    function checkNodeOperatorsPerExtraDataItemCount(uint256 _itemIndex, uint256 _nodeOperatorsCount) external view;
}
```

#### 6.3. Contract version upgrade

For correct migration to the new version of the `AccountingOracle` contract, the existing initialize methods should be updated, and the new `finalizeUpgrade_v2` external method should be implemented. See more details about Lido proxy contracts upgrade requirements in the [LIP-10](https://github.com/lidofinance/lido-improvement-proposals/blob/feat/lip-24/LIPS/lip-10.md).

```solidity
function initialize(
    address admin,
    address consensusContract,
    uint256 consensusVersion
) external {
    if (admin == address(0)) revert AdminCannotBeZero();

    uint256 lastProcessingRefSlot = _checkOracleMigration(LEGACY_ORACLE, consensusContract);
    _initialize(admin, consensusContract, consensusVersion, lastProcessingRefSlot);

    _updateContractVersion(2);
}

function initializeWithoutMigration(
    address admin,
    address consensusContract,
    uint256 consensusVersion,
    uint256 lastProcessingRefSlot
) external {
    if (admin == address(0)) revert AdminCannotBeZero();

    _initialize(admin, consensusContract, consensusVersion, lastProcessingRefSlot);

    _updateContractVersion(2);
}

function finalizeUpgrade_v2(uint256 consensusVersion) external {
    _updateContractVersion(2);
    _setConsensusVersion(consensusVersion);
}
```

### 7. `MinFirstAllocationStrategy` changes

New `memory` return value should be added to the `allocate` method of the `MinFirstAllocationStrategy` library.

```solidity
/// @notice Allocates passed maxAllocationSize among the buckets. The resulting allocation doesn't exceed the
///     capacities of the buckets. An algorithm starts filling from the least populated buckets to equalize the fill factor.
///     For example, for buckets: [9998, 70, 0], capacities: [10000, 101, 100], and maxAllocationSize: 101, the allocation happens
///     following way:
///         1. top up the bucket with index 2 on 70. Intermediate state of the buckets: [9998, 70, 70]. According to the definition,
///            the rest allocation must be proportionally split among the buckets with the same values.
///         2. top up the bucket with index 1 on 15. Intermediate state of the buckets: [9998, 85, 70].
///         3. top up the bucket with index 2 on 15. Intermediate state of the buckets: [9998, 85, 85].
///         4. top up the bucket with index 1 on 1. Nothing to distribute. The final state of the buckets: [9998, 86, 85]
/// @dev Method modifies the passed buckets array to reduce the gas costs on memory allocation.
/// @param buckets The array of current allocations in the buckets
/// @param capacities The array of capacities of the buckets
/// @param allocationSize The desired value to allocate among the buckets
/// @return allocated The total value allocated among the buckets. Can't exceed the allocationSize value
function allocate(
    uint256[] memory buckets,
    uint256[] memory capacities,
    uint256 allocationSize
) public pure returns (uint256 allocated, uint256[] memory) {
    uint256 allocatedToBestCandidate = 0;
    while (allocated < allocationSize) {
        allocatedToBestCandidate = allocateToBestCandidate(buckets, capacities, allocationSize - allocated);
        if (allocatedToBestCandidate == 0) {
            break;
        }
        allocated += allocatedToBestCandidate;
    }
    return (allocated, buckets);
}
```

## Links

- [Staking Router 2.0 on-chain code](https://github.com/lidofinance/core/tree/feat/sr-1.5);
- [New Deposit Security Module specification](https://hackmd.io/@lido/rJrTnEc2a);
- [New Validator Exit Bus Oracle specification](https://hackmd.io/@lido/BJXRTxMRp); 
- [Expanded third phase of the Accounting Oracle specification](https://hackmd.io/HCXVrrZAQNCm_IC0lYvJjw);
- [Reward distribution in curated-based modules](https://hackmd.io/@lido/HJYbVq5b0);
- [LIP-10: Proxy initializations and LidoOracle upgrade](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-10.md);
- [Backward compatibility tests of the new Staking Router contracts](https://github.com/lidofinance/sr-1.5-compatibility-tests)
