---
lip: 2
title: Oracle contract upgrade to v2
status: Work in progress
author: Denis Glotov
discussions-to: none
created: 2021-02-22
updated: 2021-12-24
---

# Oracle contract upgrade to v2

The following new features are added to the first version of the oracle contract.


## Allow the number of oracle members to be less than the quorum value.

The most likely reason for removing an oracle member is a malicious oracle. It's better to have an
oracle with no quorum (as members are added or quorum value lowered by the governance) than an
oracle with a quorum but a malicious member in it.

We implemented this by updating the restrictions (were `members.length >= _quorum`). Now the only
way to update the quorum value is with its mutator (untouched since v1):

    function setQuorum(uint256 _quorum) external auth(MANAGE_QUORUM)


## Use only one epoch per frame for oracles reporting.

In the first version of the contract, we used "min/max reportable epoch" pair to determine the range
of epochs that the contract currently accepts. This led to the overly complicated logic. In
particular, we were keeping all report pushes for epochs of the current frame until one of them
reaches quorum. And in case of oracle members updates or quorum value changes, we just invalidated
all epochs except the last one (to avoid iterating over all epochs within the frame).

So now we found it reasonable to use only the latest reported epoch for oracle reportings: when an
oracle reports a more recent epoch, we erase the current reporting (even if it did not reach a
quorum) and move to the new epoch.

⚠️ The important note here is that when we remove an oracle (with `removeOracleMember`), we also need
to remove her report from the currently accepted reports. As of now, we do not keep a mapping
between members and their reports, we just clean all existing reports and wait for the remaining
oracles to push the same epoch again.

⚠️ One more to note here is that we only allow the first epoch of the frame for reporting
(`_epochId.mod(epochsPerFrame) == 0`). This is done to prevent a malicious oracle from spoiling the
quorum by continuously reporting a new epoch.

The major change here is that we removed `gatheredEpochData` mapping. Instead, we keep
`gatheredReportsKind` array that keeps different report "kinds" gathered for the current "reportable
epoch". The report kind is a report with a counter - how many times this report was pushed by
oracles. This heavily simplified logic of `_getQuorumReport`, because in the majority of cases, we
only have 1 kind of report so we just make sure that its counter exceeded the quorum value.
`Algorithm.sol`, which used to find the majority element for the reporting, was completely removed.

The following contract storage variables are used to keep the information.

    bytes32 internal constant REPORTABLE_EPOCH_ID_POSITION =
        keccak256("lido.LidoOracle.reportableEpochId");
    bytes32 internal constant REPORTS_BITMASK_POSITION =
        keccak256("lido.LidoOracle.reoirtsBitMask");
    ReportKind[] private gatheredReportKinds;  // in place of gatheredEpochData mapping in v1


## Add calculation of staker rewards [APR][1].

To calculate the percentage of rewards for stakers, we store and provide the following data:

* `preTotalPooledEther` - total pooled ether mount, queried right before every report push to the
  Lido contract,
* `postTotalPooledEther` - the same, but queried right after the push,
* `lastCompletedEpochId` - the last epoch that we pushed the report to the Lido,
* `timeElapsed` - the time in seconds between the current epoch of push and the
  `lastCompletedEpochId`. Usually, it should be a frame long: 32 * 12 * 225 = 86400, but maybe
  multiples more in case that the previous frame didn't reach the quorum.

⚠️ It is important to note here, that we collect post/pre pair (not current/last), to avoid the
influence of new staking during the epoch.

The following contract storage variables are used to keep the information.

    bytes32 internal constant POST_COMPLETED_TOTAL_POOLED_ETHER_POSITION =
        keccak256("lido.LidoOracle.postCompletedTotalPooledEther");
    bytes32 internal constant PRE_COMPLETED_TOTAL_POOLED_ETHER_POSITION =
        keccak256("lido.LidoOracle.preCompletedTotalPooledEther");
    bytes32 internal constant LAST_COMPLETED_EPOCH_ID_POSITION =
        keccak256("lido.LidoOracle.lastCompletedEpochId");
    bytes32 internal constant TIME_ELAPSED_POSITION =
        keccak256("lido.LidoOracle.timeElapsed");

Public function was added to provide data for calculating the rewards of [stETH][2] holders.

    function getLastCompletedReportDelta()
        public view
        returns (
            uint256 postTotalPooledEther,
            uint256 preTotalPooledEther,
            uint256 timeElapsed
        )


## Sanity checks the oracles reports by configurable values.

In order to limit the misbehaving oracles impact, we want to limit oracles report change by 0.1 APR
increase in stake and 15% decrease in stake. Both values are configurable by the governance in case of
extremely unusual circumstances.

⚠️ Note that the change is evaluated after the quorum of oracles reports is reached, and not on the
individual report.

For this, we have added the following accessors and mutators:

    function getAllowedBeaconBalanceAnnualRelativeIncrease() public view returns(uint256)

    function getAllowedBeaconBalanceRelativeDecrease() public view returns(uint256)

    function setAllowedBeaconBalanceAnnualRelativeIncrease(uint256 value)
        public auth(SET_REPORT_BOUNDARIES)

    function setAllowedBeaconBalanceRelativeDecrease(uint256 value)
        public auth(SET_REPORT_BOUNDARIES)

And the logic of reporting to the Lido contract got a call to `_reportSanityChecks` that does the
following. It compares the `preTotalPooledEther` and `postTotalPooledEther` (see above) and

* if there is a profit or same, calculates the [APR][1], compares it with the upper bound. If was above,
  reverts the transaction with `ALLOWED_BEACON_BALANCE_INCREASE` code.
* if there is a loss, calculates relative decrease and compares it with the lower bound. If was
  below, reverts the transaction with `ALLOWED_BEACON_BALANCE_DECREASE` code.


## Callback function to be invoked on report pushes.

To provide the external contract with updates on report pushes (every time the quorum is reached
among oracle daemons data), we provide the following setter and getter functions.

    function setQuorumCallback(address _addr) external auth(SET_QUORUM_CALLBACK)

    function getQuorumCallback() public view returns(address)

And when the callback is set, the following function will be invoked on every report push.

    interface IQuorumCallback {
        function processLidoOracleReport(
            uint256 _postTotalPooledEther,
            uint256 _preTotalPooledEther,
            uint256 _timeElapsed) external;
    }

The arguments provided are the same as described in sections above.

The following contract storage variables are used to keep the information.

    bytes32 internal constant QUORUM_CALLBACK_POSITION = keccak256("lido.LidoOracle.quorumCallback");


## Add events to cover all states change

The goal was to cover every state change and have access to every storage variable. This is why we
added the following events.

    event BeaconSpecSet(
        uint64 epochsPerFrame,
        uint64 slotsPerEpoch,
        uint64 secondsPerSlot,
        uint64 genesisTime
    );

Reports beacon specification update by governance.

    event ReportableEpochIdUpdated(uint256 epochId)
    
Reports the new epoch that the contract is ready to accept from oracles. This happens as a result of
either a successful quorum or when some oracle reported later epoch.
    
    event BeaconReported(
        uint256 epochId,
        uint128 beaconBalance,
        uint128 beaconValidators,
        address caller
    )
    
Reports the data that the oracle pushed to the contract with `reportBeacon` call.
    
    event PostTotalShares(
         uint256 postTotalPooledEther,
         uint256 preTotalPooledEther,
         uint256 timeElapsed,
         uint256 totalShares)

Reports statistics data when the quorum happened and the resulting report was pushed to Lido. Here,
`totalShares` is grabbed from the Lido contract, see [Add calculation of staker rewards APR][3]
section above for other arguments.
         
    event AllowedBeaconBalanceAnnualRelativeIncreaseSet(uint256 value)
    event AllowedBeaconBalanceRelativeDecreaseSet(uint256 value)

Reports the updates of the threshold limits by the governance. 
See [Sanity checks the oracles reports by configurable values][4] section above for details.

    event QuorumCallbackSet(address callback);

Reports the updates of the quorum callback, [Callback function to be invoked on report pushes][5].


## Add getters for accessing the current state details.

In addition to the getters listed in sections above, the following functions provide public access
to the current reporting state.

    function getCurrentOraclesReportStatus() public view returns(uint256)

    function getCurrentReportKindSize() public view returns(uint256)

    function getCurrentReportKind(uint256 index)
        public view
        returns(
            uint128 beaconBalance,
            uint128 beaconValidators,
            uint256 count
        )



## Other changes.

Public function `initialize` was removed from v2 because it is not needed once the contract is
initialized for the first time, that happened in v1.


[1]: https://en.wikipedia.org/wiki/Annual_percentage_rate
[2]: https://lido.fi/faq
[3]: #add-calculation-of-staker-rewards-apr
[4]: #sanity-checks-the-oracles-reports-by-configurable-values
[5]: #callback-function-to-be-invoked-on-report-pushes
