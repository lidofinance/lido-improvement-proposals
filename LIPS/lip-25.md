---
lip: 25
title: Staking router 2.0
status: Proposed
author: Kirill Minenko, Aleksandr Lukin
discussions-to: <TODO>
created: 2024-05-06
updated: 2024-06-11
---

# LIP-25: Staking router 2.0 upgrade

## Simple Summary

TODO

## Motivation

TODO

> NOTE: The proposal does not make any assumptions in regards to any policies, restrictions and regulations that may be applied and only covers technical implementation.


## Mechanics

Staking router 2.0 is an upgrade of current Staking router

## Specification


### 1. IStakingModule changes

The code below assumes the Solidity v0.8.10 syntax.

```solidity
/// @title Lido's Staking Module interface
interface IStakingModule {
    /// ...
    
    /// To be changed <TODO add explanation>
    ///
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
        uint256 targetLimitMode, /// previously `bool isTargetLimitActive`
        uint256 targetValidatorsCount,
        uint256 stuckValidatorsCount,
        uint256 refundedValidatorsCount,
        uint256 stuckPenaltyEndTimestamp,
        uint256 totalExitedValidators,
        uint256 totalDepositedValidators,
        uint256 depositableValidatorsCount
    );

    /// To be added <TODO add explanation>
    ///
    /// @notice Called by StakingRouter to decrease the number of vetted keys for node operator with given id
    /// @param _nodeOperatorIds bytes packed array of the node operators id
    /// @param _vettedSigningKeysCounts bytes packed array of the new number of vetted keys for the node operators
    function decreaseVettedSigningKeysCount(
        bytes calldata _nodeOperatorIds,
        bytes calldata _vettedSigningKeysCounts
    ) external;

    /// To be changed <TODO add explanation>
    ///
    /// @notice Updates the number of the validators in the EXITED state for node operator with given id
    /// @param _nodeOperatorIds bytes packed array of the node operators id
    /// @param _exitedValidatorsCounts bytes packed array of the new number of EXITED validators for the node operators
    function updateExitedValidatorsCount(
        bytes calldata _nodeOperatorIds,
        bytes calldata _exitedValidatorsCounts /// previously `bytes calldata _stuckValidatorsCounts`
    ) external;

    /// To be changed <TODO add explanation>
    ///
    /// @notice Updates the limit of the validators that can be used for deposit
    /// @param _nodeOperatorId Id of the node operator
    /// @param _targetLimitMode taret limit mode
    /// @param _targetLimit Target limit of the node operator
    function updateTargetValidatorsLimits(
        uint256 _nodeOperatorId,
        uint256 _targetLimitMode, /// previously `bool _isTargetLimitActive`
        uint256 _targetLimit
    ) external;

    /// To be added <TODO add explanation>
    ///
    /// @dev Event to be emitted when a signing key is added to the StakingModule
    event SigningKeyAdded(uint256 indexed nodeOperatorId, bytes pubkey);

    /// To be added <TODO add explanation>
    ///
    /// @dev Event to be emitted when a signing key is removed from the StakingModule
    event SigningKeyRemoved(uint256 indexed nodeOperatorId, bytes pubkey);

    /// ...
}
```

### 2. StakingRouter changes

TODO

### 3. NodeOperatorsRegistry changes

The code below assumes the Solidity v0.4.24 syntax.

```solidity
contract NodeOperatorsRegistry is AragonApp, Versioned {
    /// ...
    /// To be added <TODO add explanation>
    ///
    event RewardDistributionStateChanged(RewardDistributionState state);

    /// Changed
    event TargetValidatorsCountChanged(uint256 indexed nodeOperatorId, uint256 targetValidatorsCount, uint256 targetLimitMode); /// previously `event TargetValidatorsCountChanged(uint256 indexed nodeOperatorId, uint256 targetValidatorsCount);`

    /// To be added <TODO add explanation>
    ///
    /// Enum to represent the state of the reward distribution process
    enum RewardDistributionState {
        TransferredToModule,      // New reward portion minted and transferred to the module
        ReadyForDistribution,     // Operators' statistics updated, reward ready for distribution
        Distributed               // Reward distributed among operators
    }
    
    /// Removed `IS_TARGET_LIMIT_ACTIVE_OFFSET` and replaced with `TARGET_LIMIT_MODE_OFFSET` 
    /// <TODO add explanation>
    ///
    /// @dev Target limit mode, allows limiting target active validators count for operator (0 = disabled, 1 = soft mode, 2 = forced mode)
    uint8 internal constant TARGET_LIMIT_MODE_OFFSET = 0;

    /// To be added <TODO add explanation>
    ///
    // bytes32 internal constant REWARD_DISTRIBUTION_STATE = keccak256("lido.NodeOperatorsRegistry.rewardDistributionState");
    bytes32 internal constant REWARD_DISTRIBUTION_STATE = 0x4ddbb0dcdc5f7692e494c15a7fca1f9eb65f31da0b5ce1c3381f6a1a1fd579b6;

    /// To be added <TODO add explanation>
    ///
    function finalizeUpgrade_v3() external {
        require(hasInitialized(), "CONTRACT_NOT_INITIALIZED");
        _checkContractVersion(2);
        _initialize_v3();
    }

    /// To be added <TODO add explanation>
    ///
    function _initialize_v3() internal {
        _setContractVersion(3);
        _updateRewardDistributionState(RewardDistributionState.Distributed);
    }

    /// To be changed <TODO add explanation>
    ///
    /// @notice Set the maximum number of validators to stake for the node operator with given id
    /// @dev Current implementation preserves invariant: depositedSigningKeysCount <= vettedSigningKeysCount <= totalSigningKeysCount.
    ///     If _vettedSigningKeysCount out of range [depositedSigningKeysCount, totalSigningKeysCount], the new vettedSigningKeysCount
    ///     value will be set to the nearest range border.
    /// @param _nodeOperatorId Node operator id to set staking limit for
    /// @param _vettedSigningKeysCount New staking limit of the node operator
    function setNodeOperatorStakingLimit(uint256 _nodeOperatorId, uint64 _vettedSigningKeysCount) external {
        _onlyExistedNodeOperator(_nodeOperatorId);
        _authP(SET_NODE_OPERATOR_LIMIT_ROLE, arr(uint256(_nodeOperatorId), uint256(_vettedSigningKeysCount)));
        _onlyCorrectNodeOperatorState(getNodeOperatorIsActive(_nodeOperatorId));

        _updateVettedSingingKeysCount(_nodeOperatorId, _vettedSigningKeysCount, true /* _allowIncrease */);
        _increaseValidatorsKeysNonce();
    }

    /// To be added <TODO add explanation>
    ///
    /// @notice Called by StakingRouter to decrease the number of vetted keys for node operator with given id
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

    /// To be added <TODO add explanation> <TODO add parameters docs>
    ///
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

    /// To be changed <TODO add explanation>
    ///
    /// @notice Called by StakingRouter to signal that stETH rewards were minted for this module.
    function onRewardsMinted(uint256 /* _totalShares */) external {
        _auth(STAKING_ROUTER_ROLE);
        _updateRewardDistributionState(RewardDistributionState.TransferredToModule);
    }

    /// To be added <TODO add explanation>
    ///
    /// @notice Permissionless method for distributing all accumulated module rewards among node operators
    /// based on the latest accounting report.
    ///
    /// @dev Rewards can be distributed after node operators' statistics are updated
    /// until the next reward is transferred to the module during the next oracle frame.
    ///
    /// ===================================== Start report frame 1 =====================================
    ///
    /// 1. Oracle first phase: Reach hash consensus.
    /// 2. Oracle second phase: Module receives rewards.
    /// 3. Oracle third phase: Operator statistics are updated.
    ///
    ///                                 ... Reward can be distributed ...
    ///
    /// =====================================  Start report frame 2  =====================================
    ///
    ///                                 ... Reward can be distributed ...
    ///                                      (if not distributed yet)
    ///
    /// 1. Oracle first phase: Reach hash consensus.
    /// 2. Oracle second phase: Module receives rewards.
    ///
    ///                                ... Reward CANNOT be distributed ...
    ///
    /// 3. Oracle third phase: Operator statistics are updated.
    ///
    ///                                 ... Reward can be distributed ...
    ///
    /// =====================================  Start report frame 3  =====================================
    function distributeReward() external {
        require(getRewardDistributionState() == RewardDistributionState.ReadyForDistribution, "DISTRIBUTION_NOT_READY");
        _updateRewardDistributionState(RewardDistributionState.Distributed);
        _distributeRewards();
    }

    /// To be changed <TODO add explanation>
    ///
    /// @notice Called by StakingRouter after it finishes updating exited and stuck validators
    /// counts for this module's node operators.
    ///
    /// Guaranteed to be called after an oracle report is applied, regardless of whether any node
    /// operator in this module has actually received any updated counts as a result of the report
    /// but given that the total number of exited validators returned from getStakingModuleSummary
    /// is the same as StakingRouter expects based on the total count received from the oracle.
    function onExitedAndStuckValidatorsCountsUpdated() external {
        _auth(STAKING_ROUTER_ROLE);
        _updateRewardDistributionState(RewardDistributionState.ReadyForDistribution);
    }
    
    /// To be changed <TODO add explanation>
    ///
    /// @notice Updates the limit of the validators that can be used for deposit by DAO
    /// @param _nodeOperatorId Id of the node operator
    /// @param _targetLimit Target limit of the node operator
    /// @param _targetLimitMode target limit mode (0 = disabled, 1 = soft mode, 2 = forced mode)
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

    /// To be changed <TODO add explanation>
    ///
    function _applyNodeOperatorLimits(uint256 _nodeOperatorId)
    internal
    returns (uint256 oldMaxSigningKeysCount, uint256 newMaxSigningKeysCount)
    {
        Packed64x4.Packed memory signingKeysStats = _loadOperatorSigningKeysStats(_nodeOperatorId);
        Packed64x4.Packed memory operatorTargetStats = _loadOperatorTargetValidatorsStats(_nodeOperatorId);

        uint256 depositedSigningKeysCount = signingKeysStats.get(TOTAL_DEPOSITED_KEYS_COUNT_OFFSET);

        // It's expected that validators don't suffer from penalties most of the time,
        // so optimistically, set the count of max validators equal to the vetted validators count.
        newMaxSigningKeysCount = signingKeysStats.get(TOTAL_VETTED_KEYS_COUNT_OFFSET);

        if (!isOperatorPenaltyCleared(_nodeOperatorId)) {
            // when the node operator is penalized zeroing its depositable validators count
            newMaxSigningKeysCount = depositedSigningKeysCount;
        } else if (operatorTargetStats.get(TARGET_LIMIT_MODE_OFFSET) != 0) { /// `IS_TARGET_LIMIT_ACTIVE_OFFSET` to be changed to `TARGET_LIMIT_MODE_OFFSET` <TODO add explanation>
            // apply target limit when it's active and the node operator is not penalized
            newMaxSigningKeysCount = Math256.max(
            // max validators count can't be less than the deposited validators count
            // even when the target limit is less than the current active validators count
                depositedSigningKeysCount,
                Math256.min(
                // max validators count can't be greater than the vetted validators count
                    newMaxSigningKeysCount,
                    // SafeMath.add() isn't used below because the sum is always
                    // less or equal to 2 * UINT64_MAX
                    signingKeysStats.get(TOTAL_EXITED_KEYS_COUNT_OFFSET)
                    + operatorTargetStats.get(TARGET_VALIDATORS_COUNT_OFFSET)
                )
            );
        }

        oldMaxSigningKeysCount = operatorTargetStats.get(MAX_VALIDATORS_COUNT_OFFSET);
        if (oldMaxSigningKeysCount != newMaxSigningKeysCount) {
            operatorTargetStats.set(MAX_VALIDATORS_COUNT_OFFSET, newMaxSigningKeysCount);
            _saveOperatorTargetValidatorsStats(_nodeOperatorId, operatorTargetStats);
        }
    }

    /// To be changed <TODO add explanation>
    ///
    function _getSigningKeysAllocationData(uint256 _keysCount)
    internal
    view
    returns (uint256 allocatedKeysCount, uint256[] memory nodeOperatorIds, uint256[] memory activeKeyCountsAfterAllocation)
    {
        uint256 activeNodeOperatorsCount = getActiveNodeOperatorsCount();
        nodeOperatorIds = new uint256[](activeNodeOperatorsCount);
        activeKeyCountsAfterAllocation = new uint256[](activeNodeOperatorsCount);
        uint256[] memory activeKeysCapacities = new uint256[](activeNodeOperatorsCount);

        uint256 activeNodeOperatorIndex;
        uint256 nodeOperatorsCount = getNodeOperatorsCount();
        uint256 maxSigningKeysCount;
        uint256 depositedSigningKeysCount;
        uint256 exitedSigningKeysCount;

        for (uint256 nodeOperatorId; nodeOperatorId < nodeOperatorsCount; ++nodeOperatorId) {
            (exitedSigningKeysCount, depositedSigningKeysCount, maxSigningKeysCount)
            = _getNodeOperator(nodeOperatorId);

            // the node operator has no available signing keys
            if (depositedSigningKeysCount == maxSigningKeysCount) continue;

            nodeOperatorIds[activeNodeOperatorIndex] = nodeOperatorId;
            activeKeyCountsAfterAllocation[activeNodeOperatorIndex] = depositedSigningKeysCount - exitedSigningKeysCount;
            activeKeysCapacities[activeNodeOperatorIndex] = maxSigningKeysCount - exitedSigningKeysCount;
            ++activeNodeOperatorIndex;
        }

        if (activeNodeOperatorIndex == 0) return (0, new uint256[](0), new uint256[](0));

        /// @dev shrink the length of the resulting arrays if some active node operators have no available keys to be deposited
        if (activeNodeOperatorIndex < activeNodeOperatorsCount) {
            assembly {
                mstore(nodeOperatorIds, activeNodeOperatorIndex)
                mstore(activeKeyCountsAfterAllocation, activeNodeOperatorIndex)
                mstore(activeKeysCapacities, activeNodeOperatorIndex)
            }
        }

        /// `activeKeyCountsAfterAllocation` to be added <TODO add explanation>
        (allocatedKeysCount, activeKeyCountsAfterAllocation) =
        MinFirstAllocationStrategy.allocate(activeKeyCountsAfterAllocation, activeKeysCapacities, _keysCount);

        /// @dev method NEVER allocates more keys than was requested
        assert(_keysCount >= allocatedKeysCount);
    }

    /// To be changed <TODO add explanation>
    ///
    function getNodeOperatorSummary(uint256 _nodeOperatorId)
    external
    view
    returns (
        uint256 targetLimitMode, /// previously `bool isTargetLimitActive`
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

        // previously `targetLimitMode = operatorTargetStats.get(TARGET_LIMIT_MODE_OFFSET);` <TODO add explanation if needed>
        targetLimitMode = operatorTargetStats.get(TARGET_LIMIT_MODE_OFFSET); 
        targetValidatorsCount = operatorTargetStats.get(TARGET_VALIDATORS_COUNT_OFFSET);
        stuckValidatorsCount = stuckPenaltyStats.get(STUCK_VALIDATORS_COUNT_OFFSET);
        refundedValidatorsCount = stuckPenaltyStats.get(REFUNDED_VALIDATORS_COUNT_OFFSET);
        stuckPenaltyEndTimestamp = stuckPenaltyStats.get(STUCK_PENALTY_END_TIMESTAMP_OFFSET);

        (totalExitedValidators, totalDepositedValidators, depositableValidatorsCount) =
        _getNodeOperatorValidatorsSummary(_nodeOperatorId);
    }

    /// To be added <TODO add explanation>
    ///
    /// @dev Get the current reward distribution state, anyone can monitor this state
    /// and distribute reward (call distributeReward method) among operators when it's `ReadyForDistribution`
    function getRewardDistributionState() public view returns (RewardDistributionState) {
        uint256 state = REWARD_DISTRIBUTION_STATE.getStorageUint256();
        return RewardDistributionState(state);
    }

    /// To be added <TODO add explanation>
    ///
    function _updateRewardDistributionState(RewardDistributionState _state) internal {
        REWARD_DISTRIBUTION_STATE.setStorageUint256(uint256(_state));
        emit RewardDistributionStateChanged(_state);
    }

    /// ...
}
```

### 4. DepositSecurityModule changes

TODO

### 5.  OracleReportSanityChecker changes

```solidity
struct LimitsList {
    /// ...

    /// `churnValidatorsPerDayLimit` to be removed and changed to 
    /// <TODO add explanation>
    ///
    /// @notice The max possible number of validators that might be reported as `exited`
    ///     per single day, depends on the Consensus Layer churn limit
    /// @dev Must fit into uint16 (<= 65_535)
    uint256 exitedValidatorsPerDayLimit;

    /// To be added <TODO add explanation>
    ///
    /// @notice The max possible number of validators that might be reported as `appeared`
    ///     per single day, limited by the max daily deposits via DepositSecurityModule in practice
    ///     isn't limited by a consensus layer (because `appeared` includes `pending`, i.e., not `activated` yet)
    /// @dev Must fit into uint16 (<= 65_535)
    uint256 appearedValidatorsPerDayLimit;
    
    /// ...
}


/// @dev The packed version of the LimitsList struct to be effectively persisted in storage
struct LimitsListPacked {
    uint16 exitedValidatorsPerDayLimit; /// previusly `uint16 churnValidatorsPerDayLimit`
    uint16 appearedValidatorsPerDayLimit; /// added
    uint16 oneOffCLBalanceDecreaseBPLimit;
    uint16 annualBalanceIncreaseBPLimit;
    uint16 simulatedShareRateDeviationBPLimit;
    uint16 maxValidatorExitRequestsPerReport;
    uint16 maxAccountingExtraDataListItemsCount;
    uint16 maxNodeOperatorsPerExtraDataItemCount;
    uint64 requestTimestampMargin;
    uint64 maxPositiveTokenRebase;
}

contract OracleReportSanityChecker is AccessControlEnumerable {
    /// ...

    /// To be added <TODO add explanation>
    ///
    bytes32 public constant EXITED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE =
    keccak256("EXITED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE");
    bytes32 public constant APPEARED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE =
    keccak256("APPEARED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE");


    /// To be changed <TODO add explanation>
    ///
    struct ManagersRoster {
        address[] allLimitsManagers;
        address[] exitedValidatorsPerDayLimitManagers; /// previously `address[] churnValidatorsPerDayLimitManagers;`
        address[] appearedValidatorsPerDayLimitManagers; /// added
        address[] oneOffCLBalanceDecreaseLimitManagers;
        address[] annualBalanceIncreaseLimitManagers;
        address[] shareRateDeviationLimitManagers;
        address[] maxValidatorExitRequestsPerReportManagers;
        address[] maxAccountingExtraDataListItemsCountManagers;
        address[] maxNodeOperatorsPerExtraDataItemCountManagers;
        address[] requestTimestampMarginManagers;
        address[] maxPositiveTokenRebaseManagers;
    }


    /// To be changed <TODO add explanation>
    ///
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
        /// `CHURN_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE` changed to `EXITED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE` <TODO add explanation>
        _grantRole(EXITED_VALIDATORS_PER_DAY_LIMIT_MANAGER_ROLE, _managersRoster.exitedValidatorsPerDayLimitManagers);
        /// To be added <TODO add explanation>
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

    /// `setChurnValidatorsPerDayLimit` renamed and change <TODO add explanation>
    ///
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

    /// To be added <TODO add explanation>
    ///
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

    /// To be changed <TODO add explanation>
    ///
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

    /// Renamed `checkAccountingExtraDataListItemsCount` to `checkExtraDataItemsCountPerTransaction` and changed <TODO add explanation>
    ///
    /// @notice Check the number of extra data list items per transaction in the accounting oracle report.
    /// @param _extraDataListItemsCount Number of items per single transaction in the accounting oracle report
    function checkExtraDataItemsCountPerTransaction(uint256 _extraDataListItemsCount)
    external
    view
    {
        uint256 limit = _limits.unpack().maxAccountingExtraDataListItemsCount;
        if (_extraDataListItemsCount > limit) {
            revert MaxAccountingExtraDataItemsCountExceeded(limit, _extraDataListItemsCount);
        }
    }

    /// To be changed <TODO add explanation>
    ///
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

    /// To be renamed from `ChurnValidatorsPerDayLimitSet` <TODO add explanation>
    event ExitedValidatorsPerDayLimitSet(uint256 exitedValidatorsPerDayLimit);

    /// To be added <TODO add explanation>
    event AppearedValidatorsPerDayLimitSet(uint256 appearedValidatorsPerDayLimit);
    /// ...


    /// renamed `churnLimit` to `appearedValidatorsLimit`
    error IncorrectAppearedValidators(uint256 appearedValidatorsLimit);

    /// renamed `churnLimit` to `exitedValudatorsLimit`
    error IncorrectExitedValidators(uint256 exitedValudatorsLimit);
}


```

### 6. AccountingOracle changes

TODO couple words about Extra data

```solidity

/// To be changed <TODO add explanation>
///
interface IOracleReportSanityChecker {
    function checkExitedValidatorsRatePerDay(uint256 _exitedValidatorsCount) external view;
    /// previously `function checkAccountingExtraDataListItemsCount(uint256 _extraDataListItemsCount) external view`
    function checkExtraDataItemsCountPerTransaction(uint256 _extraDataListItemsCount) external view;
    function checkNodeOperatorsPerExtraDataItemCount(uint256 _itemIndex, uint256 _nodeOperatorsCount) external view;
}


contract AccountingOracle is BaseOracle {
    /// ...
    
    /// To be added <TODO add explanation>
    ///
    error UnexpectedExtraDataLength();

    /// To be added <TODO add explanation>
    ///
    bytes32 internal constant ZERO_HASH = bytes32(0);

    /// To be changed <TODO add explanation>
    ///
    function initialize(
        address admin,
        address consensusContract,
        uint256 consensusVersion
    ) external {
        if (admin == address(0)) revert AdminCannotBeZero();

        uint256 lastProcessingRefSlot = _checkOracleMigration(LEGACY_ORACLE, consensusContract);
        _initialize(admin, consensusContract, consensusVersion, lastProcessingRefSlot);

        /// To be added <TODO add explanation>
        _updateContractVersion(2);
    }

    /// To be changed <TODO add explanation>
    ///
    function initializeWithoutMigration(
        address admin,
        address consensusContract,
        uint256 consensusVersion,
        uint256 lastProcessingRefSlot
    ) external {
        if (admin == address(0)) revert AdminCannotBeZero();

        _initialize(admin, consensusContract, consensusVersion, lastProcessingRefSlot);

        /// To be added <TODO add explanation>
        _updateContractVersion(2);
    }

    /// To be added <TODO add explanation>
    ///
    function finalizeUpgrade_v2(uint256 consensusVersion) external {
        _updateContractVersion(2);
        _setConsensusVersion(consensusVersion);
    }

    /// To be changed <TODO add explanation>
    ///
    /// @notice Submits report extra data in the EXTRA_DATA_FORMAT_LIST format for processing.
    ///
    /// @param data The extra data chunk with items list. See docs for the `EXTRA_DATA_FORMAT_LIST`
    ///              constant for details.
    ///
    function submitReportExtraDataList(bytes calldata data) external {
        _submitReportExtraDataList(data);
    }

    /// To be changed <TODO add explanation>
    ///
    function _checkCanSubmitExtraData(ExtraDataProcessingState memory procState, uint256 format)
    internal view
    {
        _checkMsgSenderIsAllowedToSubmitData();

        ConsensusReport memory report = _storageConsensusReport().value;

        /// `ZERO_HASH` constant instead of `bytes32(0)`
        if (report.hash == ZERO_HASH || procState.refSlot != report.refSlot) {
            revert CannotSubmitExtraDataBeforeMainData();
        }

        _checkProcessingDeadline();

        if (procState.dataFormat != format) {
            revert UnexpectedExtraDataFormat(procState.dataFormat, format);
        }
    }

    /// To be changed <TODO add explanation>
    ///
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

    /// To be changed <TODO add explanation>
    ///
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

    /// To be changed <TODO add explanation>
    ///
    function _processExtraDataItem(bytes calldata data, ExtraDataIterState memory iter) internal returns (uint256) {
        uint256 dataOffset = iter.dataOffset;
        uint256 moduleId;
        uint256 nodeOpsCount;
        uint256 nodeOpId;
        bytes calldata nodeOpIds;
        bytes calldata valuesCounts;

        if (dataOffset + 35 > data.length) {
            // has to fit at least moduleId (3 bytes), nodeOpsCount (8 bytes),
            // and data for one node operator (8 + 16 bytes), total 35 bytes
            revert InvalidExtraDataItem(iter.index);
        }

        /// @solidity memory-safe-assembly
        assembly {
        // layout at the dataOffset:
        // | 3 bytes  |   8 bytes    |  nodeOpsCount * 8 bytes  |  nodeOpsCount * 16 bytes  |
        // | moduleId | nodeOpsCount |      nodeOperatorIds     |      validatorsCounts     |
            let header := calldataload(add(data.offset, dataOffset))
            moduleId := shr(232, header)
            nodeOpsCount := and(shr(168, header), 0xffffffffffffffff)
            nodeOpIds.offset := add(data.offset, add(dataOffset, 11))
            nodeOpIds.length := mul(nodeOpsCount, 8)
        // read the 1st node operator id for checking the sorting order later
            nodeOpId := shr(192, calldataload(nodeOpIds.offset))
            valuesCounts.offset := add(nodeOpIds.offset, nodeOpIds.length)
            valuesCounts.length := mul(nodeOpsCount, 16)
            dataOffset := sub(add(valuesCounts.offset, valuesCounts.length), data.offset)
        }

        if (moduleId == 0) {
            revert InvalidExtraDataItem(iter.index);
        }

        unchecked {
        // firstly, check the sorting order between the 1st item's element and the last one of the previous item

        // | 2 bytes  | 19 bytes | 3 bytes  | 8 bytes  |
        // | itemType | 00000000 | moduleId | nodeOpId |
            uint256 sortingKey = (iter.itemType << 240) | (moduleId << 64) | nodeOpId;
            if (sortingKey <= iter.lastSortingKey) {
                revert InvalidExtraDataSortOrder(iter.index);
            }

        // secondly, check the sorting order between the rest of the elements
            uint256 tmpNodeOpId;
            for (uint256 i = 1; i < nodeOpsCount;) {
                /// @solidity memory-safe-assembly
                assembly {
                    tmpNodeOpId := shr(192, calldataload(add(nodeOpIds.offset, mul(i, 8))))
                    i := add(i, 1)
                }
                if (tmpNodeOpId <= nodeOpId) {
                    revert InvalidExtraDataSortOrder(iter.index);
                }
                nodeOpId = tmpNodeOpId;
            }

        // update the last sorting key with the last item's element
            iter.lastSortingKey = ((sortingKey >> 64) << 64) | nodeOpId;
        }

        if (dataOffset > data.length || nodeOpsCount == 0) {
            revert InvalidExtraDataItem(iter.index);
        }

        if (iter.itemType == EXTRA_DATA_TYPE_STUCK_VALIDATORS) {
            IStakingRouter(iter.stakingRouter)
            .reportStakingModuleStuckValidatorsCountByNodeOperator(moduleId, nodeOpIds, valuesCounts);
        } else {
            IStakingRouter(iter.stakingRouter)
            .reportStakingModuleExitedValidatorsCountByNodeOperator(moduleId, nodeOpIds, valuesCounts);
        }

        iter.dataOffset = dataOffset;
        return nodeOpsCount;
    }
    
    /// ...
}

```

### 7. MinFirstAllocationStrategy changes

TODO

## Security Considerations

TODO

### Links
- TODO
