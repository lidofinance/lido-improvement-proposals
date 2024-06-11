---
lip: 25
title: Staking router 2.0
status: Proposed
author: Kirill Minenko
discussions-to: https://research.lido.fi/t/lip-25-
created: 2024-05-06
updated: 2024-05-06
---

# LIP-25: Staking router 2.0 upgrade

## Simple Summary



## Motivation



> NOTE: The proposal does not make any assumptions in regards to any policies, restrictions and regulations that may be applied and only covers technical implementation.


## Mechanics

Staking router 2.0 is an upgrade of current Staking router

## Specification
We propose the following interface for `InsuranceFund`. 

The code below assumes the Solidity v0.8.10 syntax.

### IStakingModule changes

```solidity
/// @title Lido's Staking Module interface
interface IStakingModule {
    /// ...
    
    /// Changed <add>
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

    /// Added <add>
    /// @notice Called by StakingRouter to decrease the number of vetted keys for node operator with given id
    /// @param _nodeOperatorIds bytes packed array of the node operators id
    /// @param _vettedSigningKeysCounts bytes packed array of the new number of vetted keys for the node operators
    function decreaseVettedSigningKeysCount(
        bytes calldata _nodeOperatorIds,
        bytes calldata _vettedSigningKeysCounts
    ) external;

    /// Changed <add>
    /// @notice Updates the number of the validators in the EXITED state for node operator with given id
    /// @param _nodeOperatorIds bytes packed array of the node operators id
    /// @param _exitedValidatorsCounts bytes packed array of the new number of EXITED validators for the node operators
    function updateExitedValidatorsCount(
        bytes calldata _nodeOperatorIds,
        bytes calldata _exitedValidatorsCounts /// previously `bytes calldata _stuckValidatorsCounts`
    ) external;

    /// Changed <add>
    /// @notice Updates the limit of the validators that can be used for deposit
    /// @param _nodeOperatorId Id of the node operator
    /// @param _targetLimitMode taret limit mode
    /// @param _targetLimit Target limit of the node operator
    function updateTargetValidatorsLimits(
        uint256 _nodeOperatorId,
        uint256 _targetLimitMode, /// previously `bool _isTargetLimitActive`
        uint256 _targetLimit
    ) external;

    /// Added event <add>
    /// @dev Event to be emitted when a signing key is added to the StakingModule
    event SigningKeyAdded(uint256 indexed nodeOperatorId, bytes pubkey);

    /// Added event <add>
    /// @dev Event to be emitted when a signing key is removed from the StakingModule
    event SigningKeyRemoved(uint256 indexed nodeOperatorId, bytes pubkey);

    /// ...
}
```

```solidity
contract NodeOperatorsRegistry is AragonApp, Versioned {
    /// Added <add>
    event RewardDistributionStateChanged(RewardDistributionState state);

    /// Changed
    event TargetValidatorsCountChanged(uint256 indexed nodeOperatorId, uint256 targetValidatorsCount, uint256 targetLimitMode); /// previously `event TargetValidatorsCountChanged(uint256 indexed nodeOperatorId, uint256 targetValidatorsCount);`
    
    /// Added
    /// Enum to represent the state of the reward distribution process
    enum RewardDistributionState {
        TransferredToModule,      // New reward portion minted and transferred to the module
        ReadyForDistribution,     // Operators' statistics updated, reward ready for distribution
        Distributed               // Reward distributed among operators
    }
    
    /// Removed `IS_TARGET_LIMIT_ACTIVE_OFFSET` and replaced with
    /// @dev Target limit mode, allows limiting target active validators count for operator (0 = disabled, 1 = soft mode, 2 = forced mode)
    uint8 internal constant TARGET_LIMIT_MODE_OFFSET = 0;
    
    /// Added
    // bytes32 internal constant REWARD_DISTRIBUTION_STATE = keccak256("lido.NodeOperatorsRegistry.rewardDistributionState");
    bytes32 internal constant REWARD_DISTRIBUTION_STATE = 0x4ddbb0dcdc5f7692e494c15a7fca1f9eb65f31da0b5ce1c3381f6a1a1fd579b6;

    /// Add
    function finalizeUpgrade_v3() external {
        require(hasInitialized(), "CONTRACT_NOT_INITIALIZED");
        _checkContractVersion(2);
        _initialize_v3();
    }

    /// Add
    function _initialize_v3() internal {
        _setContractVersion(3);
        _updateRewardDistributionState(RewardDistributionState.Distributed);
    }
}
```

## Security Considerations
### Upgradability
!The proposed contract is non-upgradable.
### Ownership
!The ownership can be transferred but not renounced. The initial owner set upon construction is intended to be the Lido DAO Agent.

### Links
- [Lido DAO Agent](https://etherscan.io/address/0x3e40D73EB977Dc6a537aF587D48316feE66E9C8c)
- [LIP 6: In-protocol coverage application mechanism proposal
  ](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-6.md)
- [Redirecting incoming revenue stream from insurance fund to DAO treasury](https://research.lido.fi/t/redirecting-incoming-revenue-stream-from-insurance-fund-to-dao-treasury/2528)
