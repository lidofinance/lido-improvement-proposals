---
lip: 25
title: Staking Router 2.0
status: Proposed
author: Kirill Minenko, Alexander Lukin
discussions-to: <TODO>
created: 2024-05-06
updated: 2024-06-13
---

# LIP-25. Staking Router 2.0

## Simple Summary

This proposal introduces Staking Router v2.0 — the upgrade that should be made to support permissionless modules (such as [Community Staking Module](https://research.lido.fi/t/community-staking-module/5917)) and improve security and scalability of the existing [Staking Router](https://research.lido.fi/t/lip-20-staking-router/3790).

The upgrade focuses on enhancing the Deposit Security Module (DSM), the Validator Exit Bus Oracle (VEBO), the third phase of the Lido Oracle, and reward distribution mechanisms in curated-based modules.

## Motivation

In the current implementation of the Curated modules, key vetting (a process of making keys depositable) occurs through the DAO, and in particular EasyTrack motion, which the operator initiates after submitting keys. DSM changes aim to improve the process of key vetting to be able to work without the governance approval and to accommodate future permissionless modules.

The current VEBO mechanism only processes validator exits in response to user withdrawal requests. This limitation hinders the protocol's ability to manage validator exits proactively, especially for permissionless modules. From the Staking Router's side, it is proposed to consider the module's share when prioritizing validators for exit.

The introduction of the Community Staking Module, which does not limit the number of node operators, necessitates a scalable solution for the Oracle's third-phase reporting.

Current reward distribution mechanisms, tied to the third-phase finalization hook, risk exceeding block gas limits and complicate the reporting process.

## Specification


### 1. `IStakingModule` interface changes

To support the new [automated keys vetting process](https://hackmd.io/@lido/rJrTnEc2a#Automated-Vetting) in the Deposit Security Module and [Boosted Exit Requests](https://hackmd.io/@lido/rJrTnEc2a#Automated-Vetting) in the Validator Exit Bus Oracle, the `IStakingModule` interface should endure a number of changes.

All the code in this interface assumes the Solidity v0.8.9 syntax.

#### 1.1. New external methods

New `decreaseVettedSigningKeysCount` method should be added to the `IStakingModule` interface. It is supposed to be called by Staking Router to decrease the number of vetted keys for node operator with the given ID.

```solidity
/// @param _nodeOperatorIds bytes packed array of the node operators id
/// @param _vettedSigningKeysCounts bytes packed array of the new number of vetted keys for the node operators
function decreaseVettedSigningKeysCount(
    bytes calldata _nodeOperatorIds,
    bytes calldata _vettedSigningKeysCounts
) external;
```

#### 1.2. Changes in existing methods

The number of `IStakingModule` interface methods should be updated:
- The `isTargetLimitActive` boolean value of the `getNodeOperatorSummary` method should be replaced with the new `targetLimitMode` uint256 value;
- The `_stuckValidatorsCounts` param of the `updateExitedValidatorsCount` method should be replaced with the   `_exitedValidatorsCounts` param;
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

/// @notice Updates the number of the validators in the EXITED state for node operator with given id
/// @param _nodeOperatorIds bytes packed array of the node operators id
/// @param _exitedValidatorsCounts bytes packed array of the new number of EXITED validators for the node operators
function updateExitedValidatorsCount(
    bytes calldata _nodeOperatorIds,
    bytes calldata _exitedValidatorsCounts
) external;

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

#### 1.3. New events

Two new events should be added to the `IStakingModule` interface.

```solidity
/// @dev Event to be emitted when a signing key is added to the StakingModule
event SigningKeyAdded(uint256 indexed nodeOperatorId, bytes pubkey);

/// @dev Event to be emitted when a signing key is removed from the StakingModule
event SigningKeyRemoved(uint256 indexed nodeOperatorId, bytes pubkey);
```

### 2. `StakingRouter` contract changes

All the code of this contract assumes the Solidity v0.8.9 syntax.

#### 2.1. Module's share consideration for validator exits

Staking Router now should consider module's share when prioritizing validators for exit. Support of this improvement in the `StakingRouter` contract requires several changes:
- Existing external `addStakingModule` method should be updated;
- Existing external `updateStakingModule` method should be updated;
- New internal `_updateStakingModule` method should be created;
- Three new `priorityExitShareThreshold`, `maxDepositsPerBlock` and `minDepositBlockDistance` fields should be added to the `StakingModule` structure. The existing `targetShare` field should be renamed to `stakeShareLimit`;
- The `targetShare` field in the `StakingModuleCache` structure should be renamed to `stakeShareLimit`;
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

struct StakingModuleCache {
    address stakingModuleAddress;
    uint24 stakingModuleId;
    uint16 stakingModuleFee;
    uint16 treasuryFee;
    uint16 stakeShareLimit;
    StakingModuleStatus status;
    uint256 activeValidatorsCount;
    uint256 availableValidatorsCount;
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

function _loadStakingModulesCacheItem(
    uint256 _stakingModuleIndex
) internal view returns (StakingModuleCache memory cacheItem) {
    StakingModule storage stakingModuleData = _getStakingModuleByIndex(_stakingModuleIndex);

    cacheItem.stakingModuleAddress = stakingModuleData.stakingModuleAddress;
    cacheItem.stakingModuleId = stakingModuleData.id;
    cacheItem.stakingModuleFee = stakingModuleData.stakingModuleFee;
    cacheItem.treasuryFee = stakingModuleData.treasuryFee;
    cacheItem.stakeShareLimit = stakingModuleData.stakeShareLimit;
    cacheItem.status = StakingModuleStatus(stakingModuleData.status);

    (
        uint256 totalExitedValidators,
        uint256 totalDepositedValidators,
        uint256 depositableValidatorsCount
    ) = IStakingModule(cacheItem.stakingModuleAddress).getStakingModuleSummary();

    cacheItem.availableValidatorsCount = cacheItem.status == StakingModuleStatus.Active
        ? depositableValidatorsCount
        : 0;

    // the module might not receive all exited validators data yet => we need to replacing
    // the exitedValidatorsCount with the one that the staking router is aware of
    cacheItem.activeValidatorsCount =
        totalDepositedValidators -
        Math256.max(totalExitedValidators, stakingModuleData.exitedValidatorsCount);
}

function _getDepositsAllocation(
    uint256 _depositsToAllocate
)
    internal
    view
    returns (uint256 allocated, uint256[] memory allocations, StakingModuleCache[] memory stakingModulesCache)
{
    // calculate total used validators for operators
    uint256 totalActiveValidators;

    (totalActiveValidators, stakingModulesCache) = _loadStakingModulesCache();

    uint256 stakingModulesCount = stakingModulesCache.length;
    allocations = new uint256[](stakingModulesCount);
    if (stakingModulesCount > 0) {
        /// @dev new estimated active validators count
        totalActiveValidators += _depositsToAllocate;
        uint256[] memory capacities = new uint256[](stakingModulesCount);
        uint256 targetValidators;

        for (uint256 i; i < stakingModulesCount; ) {
            allocations[i] = stakingModulesCache[i].activeValidatorsCount;
            targetValidators = (stakingModulesCache[i].stakeShareLimit * totalActiveValidators) / TOTAL_BASIS_POINTS;
            capacities[i] = Math256.min(
                targetValidators,
                stakingModulesCache[i].activeValidatorsCount + stakingModulesCache[i].availableValidatorsCount
        );
        unchecked {
            ++i;
        }
      }

      (allocated, allocations) = MinFirstAllocationStrategy.allocate(allocations, capacities, _depositsToAllocate);
    }
  }
```

#### 2.2. Contract version upgrade

For correct migration to the new version of the `StakingRouter` contract, the existing `initialize` external method should be updated, and the new `finalizeUpgrade_v2` external method should be implemented. The existing `ZeroAddress` error type should be transformed to the new `ZeroAddressLido` error type. Also, the new `ZeroAddressAdmin` error type should be added.

```solidity
error ZeroAddressLido();
error ZeroAddressAdmin();

/**
* @dev proxy initialization
* @param _admin Lido DAO Aragon agent contract address
* @param _lido Lido address
* @param _withdrawalCredentials Lido withdrawal vault contract address
*/
function initialize(address _admin, address _lido, bytes32 _withdrawalCredentials) external {
    if (_admin == address(0)) revert ZeroAddressAdmin();
    if (_lido == address(0)) revert ZeroAddressLido();

    _initializeContractVersionTo(2);

    _setupRole(DEFAULT_ADMIN_ROLE, _admin);

    LIDO_POSITION.setStorageAddress(_lido);
    WITHDRAWAL_CREDENTIALS_POSITION.setStorageBytes32(_withdrawalCredentials);
    emit WithdrawalCredentialsSet(_withdrawalCredentials, msg.sender);
}

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

Support of new key vetting logic requires implementation of the new `decreaseStakingModuleVettedKeysCountByNodeOperator` external method. Also, the new `STAKING_MODULE_UNVETTING_ROLE` constant should be defined.

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

The `StackingRouter` contract should support more than two target limit modes. To implement this the following changes are needed:
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

#### 2.5. New methods for getting deposit blocks

Two new external methods for getting module's min deposit block distance and max deposits per block should be implemented.

```solidity
function getStakingModuleMinDepositBlockDistance(uint256 _stakingModuleId) external view returns (uint256) {
    return _getStakingModuleById(_stakingModuleId).minDepositBlockDistance;
}

function getStakingModuleMaxDepositsPerBlock(uint256 _stakingModuleId) external view returns (uint256) {
    return _getStakingModuleById(_stakingModuleId).maxDepositsPerBlock;
}
```

#### 2.6. New internal `_getIStakingModuleById` method

New internal `_getIStakingModuleById` method should be implemented. The following methods should use this new method to get the `IStackingModule` interface instance:
- `updateTargetValidatorsLimits`;
- `updateRefundedValidatorsCount`;
- `reportRewardsMinted`;
- `reportStakingModuleExitedValidatorsCountByNodeOperator`;
- `reportStakingModuleStuckValidatorsCountByNodeOperator`;
- `decreaseStakingModuleVettedKeysCountByNodeOperator`;
- `getAllNodeOperatorDigests`;
- `getNodeOperatorDigests`;
- `getStakingModuleNonce`.

```solidity
function _getIStakingModuleById(uint256 _stakingModuleId) internal view returns (IStakingModule) {
    return IStakingModule(_getStakingModuleAddressById(_stakingModuleId));
}

/// @notice Updates the number of the refunded validators in the staking module with the given
///     node operator id
/// @param _stakingModuleId Id of the staking module
/// @param _nodeOperatorId Id of the node operator
/// @param _refundedValidatorsCount New number of refunded validators of the node operator
function updateRefundedValidatorsCount(
    uint256 _stakingModuleId,
    uint256 _nodeOperatorId,
    uint256 _refundedValidatorsCount
) external onlyRole(STAKING_MODULE_MANAGE_ROLE) {
    _getIStakingModuleById(_stakingModuleId).updateRefundedValidatorsCount(_nodeOperatorId, _refundedValidatorsCount);
}

function reportRewardsMinted(
    uint256[] calldata _stakingModuleIds,
    uint256[] calldata _totalShares
) external onlyRole(REPORT_REWARDS_MINTED_ROLE) {
    if (_stakingModuleIds.length != _totalShares.length) {
        revert ArraysLengthMismatch(_stakingModuleIds.length, _totalShares.length);
    }

    for (uint256 i = 0; i < _stakingModuleIds.length; ) {
        if (_totalShares[i] > 0) {
            try _getIStakingModuleById(_stakingModuleIds[i]).onRewardsMinted(_totalShares[i]) {} catch (
                bytes memory lowLevelRevertData
            ) {
                /// @dev This check is required to prevent incorrect gas estimation of the method.
                ///      Without it, Ethereum nodes that use binary search for gas estimation may
                ///      return an invalid value when the onRewardsMinted() reverts because of the
                ///      "out of gas" error. Here we assume that the onRewardsMinted() method doesn't
                ///      have reverts with empty error data except "out of gas".
                if (lowLevelRevertData.length == 0) revert UnrecoverableModuleError();
                emit RewardsMintedReportFailed(_stakingModuleIds[i], lowLevelRevertData);
            }
        }
        unchecked {
            ++i;
        }
    }
}

/// @notice Updates exited validators counts per node operator for the staking module with
/// the specified id.
///
/// See the docs for `updateExitedValidatorsCountByStakingModule` for the description of the
/// overall update process.
///
/// @param _stakingModuleId The id of the staking modules to be updated.
/// @param _nodeOperatorIds Ids of the node operators to be updated.
/// @param _exitedValidatorsCounts New counts of exited validators for the specified node operators.
///
function reportStakingModuleExitedValidatorsCountByNodeOperator(
    uint256 _stakingModuleId,
    bytes calldata _nodeOperatorIds,
    bytes calldata _exitedValidatorsCounts
) external onlyRole(REPORT_EXITED_VALIDATORS_ROLE) {
    _checkValidatorsByNodeOperatorReportData(_nodeOperatorIds, _exitedValidatorsCounts);
    _getIStakingModuleById(_stakingModuleId).updateExitedValidatorsCount(_nodeOperatorIds, _exitedValidatorsCounts);
}

/// @notice Updates stuck validators counts per node operator for the staking module with
/// the specified id.
///
/// See the docs for `updateExitedValidatorsCountByStakingModule` for the description of the
/// overall update process.
///
/// @param _stakingModuleId The id of the staking modules to be updated.
/// @param _nodeOperatorIds Ids of the node operators to be updated.
/// @param _stuckValidatorsCounts New counts of stuck validators for the specified node operators.
///
function reportStakingModuleStuckValidatorsCountByNodeOperator(
    uint256 _stakingModuleId,
    bytes calldata _nodeOperatorIds,
    bytes calldata _stuckValidatorsCounts
) external onlyRole(REPORT_EXITED_VALIDATORS_ROLE) {
    _checkValidatorsByNodeOperatorReportData(_nodeOperatorIds, _stuckValidatorsCounts);
    _getIStakingModuleById(_stakingModuleId).updateStuckValidatorsCount(_nodeOperatorIds, _stuckValidatorsCounts);
}

/// @notice Returns node operator digest for each node operator registered in the given staking module
/// @param _stakingModuleId id of the staking module to return data for
/// @dev WARNING: This method is not supposed to be used for onchain calls due to high gas costs
///     for data aggregation
function getAllNodeOperatorDigests(uint256 _stakingModuleId) external view returns (NodeOperatorDigest[] memory) {
    return
        getNodeOperatorDigests(_stakingModuleId, 0, _getIStakingModuleById(_stakingModuleId).getNodeOperatorsCount());
}

/// @notice Returns node operator digest for passed node operator ids in the given staking module
/// @param _stakingModuleId id of the staking module where node operators registered
/// @param _offset node operators offset starting with 0
/// @param _limit the max number of node operators to return
/// @dev WARNING: This method is not supposed to be used for onchain calls due to high gas costs
///     for data aggregation
function getNodeOperatorDigests(
    uint256 _stakingModuleId,
    uint256 _offset,
    uint256 _limit
) public view returns (NodeOperatorDigest[] memory) {
    return
        getNodeOperatorDigests(
            _stakingModuleId,
            _getIStakingModuleById(_stakingModuleId).getNodeOperatorIds(_offset, _limit)
        );
}

/// @notice Returns node operator digest for a slice of node operators registered in the given
///     staking module
/// @param _stakingModuleId id of the staking module where node operators registered
/// @param _nodeOperatorIds ids of the node operators to return data for
/// @dev WARNING: This method is not supposed to be used for onchain calls due to high gas costs
///     for data aggregation
function getNodeOperatorDigests(
    uint256 _stakingModuleId,
    uint256[] memory _nodeOperatorIds
) public view returns (NodeOperatorDigest[] memory digests) {
    IStakingModule stakingModule = _getIStakingModuleById(_stakingModuleId);
    digests = new NodeOperatorDigest[](_nodeOperatorIds.length);
    for (uint256 i = 0; i < _nodeOperatorIds.length; ++i) {
        digests[i] = NodeOperatorDigest({
            id: _nodeOperatorIds[i],
            isActive: stakingModule.getNodeOperatorIsActive(_nodeOperatorIds[i]),
            summary: getNodeOperatorSummary(_stakingModuleId, _nodeOperatorIds[i])
        });
    }
}

function getStakingModuleNonce(uint256 _stakingModuleId) external view returns (uint256) {
    return _getIStakingModuleById(_stakingModuleId).getNonce();
}
```

#### 2.7. Excluded methods for module pausing

Methods, events and constants related to Staking Module pausing logic should be removed from the `StakingRouter` contract:
- `StakingModuleNotPaused` error type;
- `STAKING_MODULE_PAUSE_ROLE` and `STAKING_MODULE_RESUME_ROLE` constants;
- `pauseStakingModule` and `resumeStakingModule` methods.

### 3. `NodeOperatorsRegistry` contract changes

All the code of this contract assumes the Solidity v0.4.24 syntax.

#### 3.1. Contract initialization engine

The `initialize` method should now call the new internal `_initialize_v3` method. Also, the new `finalizeUpgrade_v3` external method should be implemented.

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

The new `decreaseVettedSigningKeysCount` method should be implemented. It is called by the Staking Router to decrease the number of vetted keys for the node operator with the given ID.

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
```

Also, the existing `setNodeOperatorStakingLimit` external method and the  `_updateVettedSingingKeysCount` internal method should be updated.

```solidity
function setNodeOperatorStakingLimit(uint256 _nodeOperatorId, uint64 _vettedSigningKeysCount) external {
    _onlyExistedNodeOperator(_nodeOperatorId);
    _authP(SET_NODE_OPERATOR_LIMIT_ROLE, arr(uint256(_nodeOperatorId), uint256(_vettedSigningKeysCount)));
    _onlyCorrectNodeOperatorState(getNodeOperatorIsActive(_nodeOperatorId));

    _updateVettedSingingKeysCount(_nodeOperatorId, _vettedSigningKeysCount, true /* _allowIncrease */);
    _increaseValidatorsKeysNonce();
}

function _updateVettedSingingKeysCount(
    uint256 _nodeOperatorId,
    uint256 _vettedSigningKeysCount,
    bool _allowIncrease
) internal {
    Packed64x4.Packed memory signingKeysStats = _loadOperatorSigningKeysStats(_nodeOperatorId);
    uint256 vettedSigningKeysCountBefore = signingKeysStats.get(TOTAL_VETTED_KEYS_COUNT_OFFSET);
    uint256 depositedSigningKeysCount = signingKeysStats.get(TOTAL_DEPOSITED_KEYS_COUNT_OFFSET);
    uint256 totalSigningKeysCount = signingKeysStats.get(TOTAL_KEYS_COUNT_OFFSET);

    uint256 vettedSigningKeysCountAfter = Math256.min(
        totalSigningKeysCount, Math256.max(_vettedSigningKeysCount, depositedSigningKeysCount)
    );

    if (vettedSigningKeysCountAfter == vettedSigningKeysCountBefore) return;

    require(
        _allowIncrease || vettedSigningKeysCountAfter < vettedSigningKeysCountBefore,
        "VETTED_KEYS_COUNT_INCREASED"
    );

    signingKeysStats.set(TOTAL_VETTED_KEYS_COUNT_OFFSET, vettedSigningKeysCountAfter);
    _saveOperatorSigningKeysStats(_nodeOperatorId, signingKeysStats);

    emit VettedSigningKeysCountChanged(_nodeOperatorId, vettedSigningKeysCountAfter);

    _updateSummaryMaxValidatorsCount(_nodeOperatorId);
}
```

#### 3.3. New reward distribution engine

The new external permissionless `distributeReward` method should be implemented. This method should distribute all accumulated module rewards among node operators based on the latest accounting report.

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


Also, the new `_updateRewardDistributionState` internal method should be implemented. It should emit the new `RewardDistributionStateChanged` event. This method should be called in a bunch of new and existing methods, such as `_initialize_v3`, `onRewardsMinted`, `updateRefundedValidatorsCount`, `distributeReward`, and `onExitedAndStuckValidatorsCountsUpdated`.

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

function onExitedAndStuckValidatorsCountsUpdated() external {
    _auth(STAKING_ROUTER_ROLE);
    _updateRewardDistributionState(RewardDistributionState.ReadyForDistribution);
}
```

#### 3.4. New target limit modes

Now it should be possible to update the target validators limit using 3 modes instead of 2: "disabled", "soft mode", and "forced mode". The `updateTargetValidatorsLimits` public method should be updated to support this change. The existing `TargetValidatorsCountChanged` event also should be updated.

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

The keys vetting logic should be changed to support optimistic vetting. This requires a bunch of new events and methods to be defined:
- New `unvetSigningKeys` external method that unvets signing keys for the given node operators;
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

#### 4.4. Changes in `canDeposit` method

New optimistic keys vetting and deposit pause logic require the `canDeposit` method to be changed. It also requires implementation of the new `isMinDepositDistancePassed` external method and the new `_isMinDepositDistancePassed` internal method.

```solidity
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

#### 4.5. Changes in `depositBufferedEther` method

New deposit logic requires changes in the `depositBufferedEther` method. It also requires changes in the `_verifyAttestSignatures` internal method (former `_verifySignatures` method) and the new `ModuleNonceChanged` error type.

```solidity
error ModuleNonceChanged();

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

function _verifyAttestSignatures(
    bytes32 depositRoot,
    uint256 blockNumber,
    bytes32 blockHash,
    uint256 stakingModuleId,
    uint256 nonce,
    Signature[] memory sigs
) internal view {
    bytes32 msgHash = keccak256(
        abi.encodePacked(ATTEST_MESSAGE_PREFIX, blockNumber, blockHash, depositRoot, stakingModuleId, nonce)
    );

    address prevSignerAddr = address(0);

    for (uint256 i = 0; i < sigs.length; ) {
        address signerAddr = ECDSA.recover(msgHash, sigs[i].r, sigs[i].vs);
        if (!_isGuardian(signerAddr)) revert InvalidSignature();
        if (signerAddr <= prevSignerAddr) revert SignaturesNotSorted();
        prevSignerAddr = signerAddr;

        unchecked {
            ++i;
        }
    }
}
```

#### 4.6. New methods for getting the last deposit block

Now it should be possible to get the block number of the last deposit. This requires the following changes:
- New `getLastDepositBlock` external method;
- New `_setLastDepositBlock` internal method;
- New `LastDepositBlockChanged` event;
- New `lastDepositBlock` field.

```solidity
event LastDepositBlockChanged(uint256 newValue);

uint256 internal lastDepositBlock;

/**
* @return lastDepositBlock The block number of the last deposit.
*/
function getLastDepositBlock() external view returns (uint256) {
    return lastDepositBlock;
}

function _setLastDepositBlock(uint256 newValue) internal {
    lastDepositBlock = newValue;
    emit LastDepositBlockChanged(newValue);
}
```

#### 4.7. Changes in contract constructor

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

#### 4.8. New code version constant

New code version constants that helps to distinguish contract interfaces should be added to the contract.

```solidity
uint256 public constant VERSION = 3;
```

### 5. `OracleReportSanityChecker` contract changes

To support the new multi-transactional third phase, two oracle sanity checker limits should be reconsidered:
- Existing `churnValidatorsPerDayLimit` should be replaced with the new `exitedValidatorsPerDayLimit`. It is the maximum possible number of validators that might be reported as `exited` per single day, depends on the Consensus Layer churn limit.
- New `appearedValidatorsPerDayLimit` should be added. It is the maximum possible number of validators that might be reported as `appeared` per single day.

All related structures, constants and methods of the `OracleReportSanityChecker` contract should be updated to support these two new limits.

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

/// @dev The packed version of the LimitsList struct to be effectively persisted in storage
struct LimitsListPacked {
    uint16 exitedValidatorsPerDayLimit;
    uint16 appearedValidatorsPerDayLimit;
    uint16 oneOffCLBalanceDecreaseBPLimit;
    uint16 annualBalanceIncreaseBPLimit;
    uint16 simulatedShareRateDeviationBPLimit;
    uint16 maxValidatorExitRequestsPerReport;
    uint16 maxAccountingExtraDataListItemsCount;
    uint16 maxNodeOperatorsPerExtraDataItemCount;
    uint64 requestTimestampMargin;
    uint64 maxPositiveTokenRebase;
}

bytes32 public constant EXITED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE =
        keccak256("EXITED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE");

bytes32 public constant APPEARED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE =
        keccak256("APPEARED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE");

struct ManagersRoster {
    address[] allLimitsManagers;
    address[] exitedValidatorsPerDayLimitManagers;
    address[] appearedValidatorsPerDayLimitManagers;
    address[] oneOffCLBalanceDecreaseLimitManagers;
    address[] annualBalanceIncreaseLimitManagers;
    address[] shareRateDeviationLimitManagers;
    address[] maxValidatorExitRequestsPerReportManagers;
    address[] maxAccountingExtraDataListItemsCountManagers;
    address[] maxNodeOperatorsPerExtraDataItemCountManagers;
    address[] requestTimestampMarginManagers;
    address[] maxPositiveTokenRebaseManagers;
}

/// @param _lidoLocator address of the LidoLocator instance
/// @param _admin address to grant DEFAULT_ADMIN_ROLE of the AccessControl contract
/// @param _limitsList initial values to be set for the limits list
/// @param _managersRoster list of the address to grant permissions for granular limits management
constructor(
    address _lidoLocator,
    address _admin,
    LimitsList memory _limitsList,
    ManagersRoster memory _managersRoster
) {
    if (_admin == address(0)) revert AdminCannotBeZero();
    LIDO_LOCATOR = ILidoLocator(_lidoLocator);

    _updateLimits(_limitsList);

    _grantRole(DEFAULT_ADMIN_ROLE, _admin);
    _grantRole(ALL_LIMITS_MANAGER_ROLE, _managersRoster.allLimitsManagers);
    _grantRole(EXITED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE, _managersRoster.exitedValidatorsPerDayLimitManagers);
    _grantRole(APPEARED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE,
                   _managersRoster.appearedValidatorsPerDayLimitManagers);
    _grantRole(ONE_OFF_CL_BALANCE_DECREASE_LIMIT_MANAGER_ROLE,
                   _managersRoster.oneOffCLBalanceDecreaseLimitManagers);
    _grantRole(ANNUAL_BALANCE_INCREASE_LIMIT_MANAGER_ROLE, _managersRoster.annualBalanceIncreaseLimitManagers);
    _grantRole(MAX_POSITIVE_TOKEN_REBASE_MANAGER_ROLE, _managersRoster.maxPositiveTokenRebaseManagers);
    _grantRole(MAX_VALIDATOR_EXIT_REQUESTS_PER_REPORT_ROLE,
                   _managersRoster.maxValidatorExitRequestsPerReportManagers);
    _grantRole(MAX_ACCOUNTING_EXTRA_DATA_LIST_ITEMS_COUNT_ROLE,
                   _managersRoster.maxAccountingExtraDataListItemsCountManagers);
    _grantRole(MAX_NODE_OPERATORS_PER_EXTRA_DATA_ITEM_COUNT_ROLE,
                   _managersRoster.maxNodeOperatorsPerExtraDataItemCountManagers);
    _grantRole(SHARE_RATE_DEVIATION_LIMIT_MANAGER_ROLE, _managersRoster.shareRateDeviationLimitManagers);
    _grantRole(REQUEST_TIMESTAMP_MARGIN_MANAGER_ROLE, _managersRoster.requestTimestampMarginManagers);
}

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

function _checkAppearedValidatorsChurnLimit(
    LimitsList memory _limitsList,
    uint256 _appearedValidators,
    uint256 _timeElapsed
) internal pure {
    if (_timeElapsed == 0) {
        _timeElapsed = DEFAULT_TIME_ELAPSED;
    }

    uint256 appearedLimit = (_limitsList.appearedValidatorsPerDayLimit * _timeElapsed) / SECONDS_PER_DAY;

    if (_appearedValidators > appearedLimit) revert IncorrectAppearedValidators(_appearedValidators);
}

function _updateLimits(LimitsList memory _newLimitsList) internal {
    LimitsList memory _oldLimitsList = _limits.unpack();
    if (_oldLimitsList.exitedValidatorsPerDayLimit != _newLimitsList.exitedValidatorsPerDayLimit) {
        _checkLimitValue(_newLimitsList.exitedValidatorsPerDayLimit, 0, type(uint16).max);
        emit ExitedValidatorsPerDayLimitSet(_newLimitsList.exitedValidatorsPerDayLimit);
    }
    if (_oldLimitsList.appearedValidatorsPerDayLimit != _newLimitsList.appearedValidatorsPerDayLimit) {
        _checkLimitValue(_newLimitsList.appearedValidatorsPerDayLimit, 0, type(uint16).max);
        emit AppearedValidatorsPerDayLimitSet(_newLimitsList.appearedValidatorsPerDayLimit);
    }
    if (_oldLimitsList.oneOffCLBalanceDecreaseBPLimit != _newLimitsList.oneOffCLBalanceDecreaseBPLimit) {
        _checkLimitValue(_newLimitsList.oneOffCLBalanceDecreaseBPLimit, 0, MAX_BASIS_POINTS);
        emit OneOffCLBalanceDecreaseBPLimitSet(_newLimitsList.oneOffCLBalanceDecreaseBPLimit);
    }
    if (_oldLimitsList.annualBalanceIncreaseBPLimit != _newLimitsList.annualBalanceIncreaseBPLimit) {
        _checkLimitValue(_newLimitsList.annualBalanceIncreaseBPLimit, 0, MAX_BASIS_POINTS);
        emit AnnualBalanceIncreaseBPLimitSet(_newLimitsList.annualBalanceIncreaseBPLimit);
    }
    if (_oldLimitsList.simulatedShareRateDeviationBPLimit != _newLimitsList.simulatedShareRateDeviationBPLimit) {
        _checkLimitValue(_newLimitsList.simulatedShareRateDeviationBPLimit, 0, MAX_BASIS_POINTS);
        emit SimulatedShareRateDeviationBPLimitSet(_newLimitsList.simulatedShareRateDeviationBPLimit);
    }
    if (_oldLimitsList.maxValidatorExitRequestsPerReport != _newLimitsList.maxValidatorExitRequestsPerReport) {
        _checkLimitValue(_newLimitsList.maxValidatorExitRequestsPerReport, 0, type(uint16).max);
        emit MaxValidatorExitRequestsPerReportSet(_newLimitsList.maxValidatorExitRequestsPerReport);
    }
    if (_oldLimitsList.maxAccountingExtraDataListItemsCount != _newLimitsList.maxAccountingExtraDataListItemsCount) {
        _checkLimitValue(_newLimitsList.maxAccountingExtraDataListItemsCount, 0, type(uint16).max);
        emit MaxAccountingExtraDataListItemsCountSet(_newLimitsList.maxAccountingExtraDataListItemsCount);
    }
    if (_oldLimitsList.maxNodeOperatorsPerExtraDataItemCount != _newLimitsList.maxNodeOperatorsPerExtraDataItemCount) {
        _checkLimitValue(_newLimitsList.maxNodeOperatorsPerExtraDataItemCount, 0, type(uint16).max);
        emit MaxNodeOperatorsPerExtraDataItemCountSet(_newLimitsList.maxNodeOperatorsPerExtraDataItemCount);
    }
    if (_oldLimitsList.requestTimestampMargin != _newLimitsList.requestTimestampMargin) {
        _checkLimitValue(_newLimitsList.requestTimestampMargin, 0, type(uint64).max);
        emit RequestTimestampMarginSet(_newLimitsList.requestTimestampMargin);
    }
    if (_oldLimitsList.maxPositiveTokenRebase != _newLimitsList.maxPositiveTokenRebase) {
        _checkLimitValue(_newLimitsList.maxPositiveTokenRebase, 1, type(uint64).max);
        emit MaxPositiveTokenRebaseSet(_newLimitsList.maxPositiveTokenRebase);
    }
    _limits = _newLimitsList.pack();
}

event ExitedValidatorsPerDayLimitSet(uint256 exitedValidatorsPerDayLimit);
event AppearedValidatorsPerDayLimitSet(uint256 appearedValidatorsPerDayLimit);

library LimitsListPacker {
    function pack(LimitsList memory _limitsList) internal pure returns (LimitsListPacked memory res) {
        res.exitedValidatorsPerDayLimit = SafeCast.toUint16(_limitsList.exitedValidatorsPerDayLimit);
        res.appearedValidatorsPerDayLimit = SafeCast.toUint16(_limitsList.appearedValidatorsPerDayLimit);
        res.oneOffCLBalanceDecreaseBPLimit = _toBasisPoints(_limitsList.oneOffCLBalanceDecreaseBPLimit);
        res.annualBalanceIncreaseBPLimit = _toBasisPoints(_limitsList.annualBalanceIncreaseBPLimit);
        res.simulatedShareRateDeviationBPLimit = _toBasisPoints(_limitsList.simulatedShareRateDeviationBPLimit);
        res.requestTimestampMargin = SafeCast.toUint64(_limitsList.requestTimestampMargin);
        res.maxPositiveTokenRebase = SafeCast.toUint64(_limitsList.maxPositiveTokenRebase);
        res.maxValidatorExitRequestsPerReport = SafeCast.toUint16(_limitsList.maxValidatorExitRequestsPerReport);
        res.maxAccountingExtraDataListItemsCount = SafeCast.toUint16(_limitsList.maxAccountingExtraDataListItemsCount);
        res.maxNodeOperatorsPerExtraDataItemCount = SafeCast.toUint16(_limitsList.maxNodeOperatorsPerExtraDataItemCount);
    }

    function _toBasisPoints(uint256 _value) private pure returns (uint16) {
        require(_value <= MAX_BASIS_POINTS, "BASIS_POINTS_OVERFLOW");
        return uint16(_value);
    }
}

library LimitsListUnpacker {
    function unpack(LimitsListPacked memory _limitsList) internal pure returns (LimitsList memory res) {
        res.exitedValidatorsPerDayLimit = _limitsList.exitedValidatorsPerDayLimit;
        res.appearedValidatorsPerDayLimit = _limitsList.appearedValidatorsPerDayLimit;
        res.oneOffCLBalanceDecreaseBPLimit = _limitsList.oneOffCLBalanceDecreaseBPLimit;
        res.annualBalanceIncreaseBPLimit = _limitsList.annualBalanceIncreaseBPLimit;
        res.simulatedShareRateDeviationBPLimit = _limitsList.simulatedShareRateDeviationBPLimit;
        res.requestTimestampMargin = _limitsList.requestTimestampMargin;
        res.maxPositiveTokenRebase = _limitsList.maxPositiveTokenRebase;
        res.maxValidatorExitRequestsPerReport = _limitsList.maxValidatorExitRequestsPerReport;
        res.maxAccountingExtraDataListItemsCount = _limitsList.maxAccountingExtraDataListItemsCount;
        res.maxNodeOperatorsPerExtraDataItemCount = _limitsList.maxNodeOperatorsPerExtraDataItemCount;
    }
}
```

### 6. `AccountingOracle` contract changes

All the code of this contract assumes the Solidity v0.8.9 syntax.

#### 6.1. Multi-transactional third phase

To support multi-transactional third phase of the Accounting Oracle, a number of changes should be implemented in the `AccountingOracle` contract:
- The `IOracleReportSanityChecker` call should be removed from the existing `_handleConsensusReportData` internal method;
- Implementation of the `_submitReportExtraDataList` and `_processExtraDataItems` internal methods should be changed;
- Second and third phases of the `_processExtraDataItem` internal methods should be updated;
- Existing `ExtraDataListOnlySupportsSingleTx` error type should be replaced with the new `UnexpectedExtraDataLength` error type.

```solidity
error UnexpectedExtraDataLength();

bytes32 internal constant ZERO_HASH = bytes32(0);

function _handleConsensusReportData(ReportData calldata data, uint256 prevRefSlot) internal {
    if (data.extraDataFormat == EXTRA_DATA_FORMAT_EMPTY) {
        if (data.extraDataHash != ZERO_HASH) {
            revert UnexpectedExtraDataHash(ZERO_HASH, data.extraDataHash);
        }
        if (data.extraDataItemsCount != 0) {
            revert UnexpectedExtraDataItemsCount(0, data.extraDataItemsCount);
        }
    } else {
        if (data.extraDataFormat != EXTRA_DATA_FORMAT_LIST) {
            revert UnsupportedExtraDataFormat(data.extraDataFormat);
        }
        if (data.extraDataItemsCount == 0) {
            revert ExtraDataItemsCountCannotBeZeroForNonEmptyData();
        }
        if (data.extraDataHash == ZERO_HASH) {
            revert ExtraDataHashCannotBeZeroForNonEmptyData();
        }
    }

    ILegacyOracle(LEGACY_ORACLE).handleConsensusLayerReport(
        data.refSlot,
        data.clBalanceGwei * 1e9,
        data.numValidators
    );

    uint256 slotsElapsed = data.refSlot - prevRefSlot;

    IStakingRouter stakingRouter = IStakingRouter(LOCATOR.stakingRouter());
    IWithdrawalQueue withdrawalQueue = IWithdrawalQueue(LOCATOR.withdrawalQueue());

    _processStakingRouterExitedValidatorsByModule(
        stakingRouter,
        data.stakingModuleIdsWithNewlyExitedValidators,
        data.numExitedValidatorsByStakingModule,
        slotsElapsed
    );

    withdrawalQueue.onOracleReport(
        data.isBunkerMode,
        GENESIS_TIME + prevRefSlot * SECONDS_PER_SLOT,
        GENESIS_TIME + data.refSlot * SECONDS_PER_SLOT
    );

    ILido(LIDO).handleOracleReport(
        GENESIS_TIME + data.refSlot * SECONDS_PER_SLOT,
        slotsElapsed * SECONDS_PER_SLOT,
        data.numValidators,
        data.clBalanceGwei * 1e9,
        data.withdrawalVaultBalance,
        data.elRewardsVaultBalance,
        data.sharesRequestedToBurn,
        data.withdrawalFinalizationBatches,
        data.simulatedShareRate
    );

    _storageExtraDataProcessingState().value = ExtraDataProcessingState({
        refSlot: data.refSlot.toUint64(),
        dataFormat: data.extraDataFormat.toUint16(),
        submitted: false,
        dataHash: data.extraDataHash,
        itemsCount: data.extraDataItemsCount.toUint16(),
        itemsProcessed: 0,
        lastSortingKey: 0
    });
}

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

As the number of assumptions were changed, sanity checkers for Accounting Oracle should be updated as well:
- The `checkAccountingExtraDataListItemsCount` method of the `IOracleReportSanityChecker` interface should be replaced by the `checkExtraDataItemsCountPerTransaction` method with the same signature.

```solidity
interface IOracleReportSanityChecker {
    function checkExitedValidatorsRatePerDay(uint256 _exitedValidatorsCount) external view;
    function checkExtraDataItemsCountPerTransaction(uint256 _extraDataListItemsCount) external view;
    function checkNodeOperatorsPerExtraDataItemCount(uint256 _itemIndex, uint256 _nodeOperatorsCount) external view;
}
```

#### 6.3. Contract version upgrade

For correct migration to the new version of the `AccountingOracle` contract, the existing initialize methods should be updated, and the new `finalizeUpgrade_v2` external method should be implemented.

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
