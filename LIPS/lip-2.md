---
lip: 2
title: Oracle contract upgrade to v2
status: Work in progress
author: Denis Glotov
discussions-to: none
created: 2021-02-22
updated: 2021-02-24
---

# Oracle contract upgrade to v2

The following changes are to be made to the first mainnet version of the oracle contract.


## Change the meaning of 'quorum'.

In the first version, the quorum value denoted the minimum number of oracles needed to successfully report
results.

The proposed change is that the governance-controlled 'quorum' value means the minimum number of exactly the same
reports needed to finalize this epoch and report this report to Lido. The reason for that change is that all non-byzantine oracles need to report the exact same value and if there are conflicting reports in the frame, it means some oracles are faulty or malicious. With an old system it took a majority of quiorum of faulty oracles to push their values (e.g. 2 out of 3), with new system it takes a full quorum of them (3 out of 3). 


For example, if the quorum value is `5` and suppose the oracles report consequently: `100`, `100`,
`101`, `0`, `100`, `100`, `100`: after the last, the report `100` wins because it was pushed 5
times. So it is pushed to Lido, epoch closes and no more reports for this epoch are accepted.


## Allow the number of oracle members to be less than the quorum value.

Current oracle removal lever (`removeOracleMember`) doesn't allow to remove oracle members if it
puts a number of members below quorum.

The most likely reason for removing an oracle member is a malicious oracle. It's better to have an
oracle with no quorum (as members are added or quorum value lowered by the voting process) than an
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

:warning: The important note here is that when we remove an oracle (with `removeOracleMember`), we also need
to remove her report from the currently accepted reports. As of now, we do not keep a mapping
between members and their reports, we just clean all existing reports and wait for the remaining
oracles to push the same epoch again.

:warning: One more to note here is that we only allow the first epoch of the frame for reporting
(`_epochId.mod(epochsPerFrame) == 0`). This is done to prevent a malicious oracle from spoiling the
quorum by continuously reporting a new epoch.

The major change here is that we removed `gatheredEpochData` mapping. Instead, we keep
`gatheredReportsKind` array that keeps different report "kinds" gathered for the current "reportable
epoch". The report kind is a report with a counter - how many times this report was pushed by
oracles. This heavily simplified logic of `_getQuorumReport`, because in the majority of cases, we
only have 1 kind of report so we just make sure that its counter exceeded the quorum value.
`Algorithm.sol`, which used to find the majority element for the reporting, was completely removed.

This change does not impact fair weather operations at all; the only setting where it is inferior to
the origninal procedure is a very specific kind of theoretical long-term eth2 turbulence where eth2
finality makes progress but lags behind last slot and that lag is 24h+ for a long time. It's been
never observed in the testnets and experts on eth2 consensus say it's a very convoluted scenario. So
we decided to axe it, and if it happens in the wild (which it won't), we can upgrade oracle back to
handle it better.

The following contract storage variables are used to keep the information.

    bytes32 internal constant REPORTABLE_EPOCH_ID_POSITION =
        keccak256("lido.LidoOracle.reportableEpochId");
    bytes32 internal constant REPORTS_BITMASK_POSITION =
        keccak256("lido.LidoOracle.reoirtsBitMask");
    ReportKind[] private gatheredReportKinds;  // in place of gatheredEpochData mapping in v1

:warning: Note that we're removing the report mapping `gatheredEpochData` and putting `ReportKind[]
private gatheredReportKinds;` in its place in the contract storage.


## Add calculation of staker rewards [APR][1].

To calculate the percentage of rewards for stakers, we store and provide the following data:

* `preTotalPooledEther` - total pooled ether mount, queried right before every report push to the
  Lido contract,
* `postTotalPooledEther` - the same, but queried right after the push,
* `lastCompletedEpochId` - the last epoch that we pushed the report to the Lido,
* `timeElapsed` - the time in seconds between the current epoch of push and the
  `lastCompletedEpochId`. Usually, it should be a frame long: 32 * 12 * 225 = 86400, but maybe
  multiples more in case that the previous frame didn't reach the quorum.

:warning: It is important to note here, that we collect post/pre pair (not current/last), to avoid
the influence of new staking during the epoch.

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
To calculate APR, use the following formula:

APR = (postTotalPooledEther - preTotalPooledEther)*secondsInYear/(preTotalPooledEther*timeElapsed)


## Sanity checks the oracles reports by configurable values.

In order to limit the misbehaving oracles impact, we want to limit oracles report change by 10% APR
increase in stake and 5% decrease in stake. Both values are configurable by the governance in case of
extremely unusual circumstances.

:warning: Note that the change is evaluated after the quorum of oracles reports is reached, and not
on the individual report.

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
among oracle daemons data), we provide the following setter and getter functions. It might be needed to implement some updates to the external contracts that should happen at the same tx the rebase happens (e.g. adjusting uniswap v2 pools to reflect the rebase).

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

It can be reasonably argued that these events and getters belong to Lido app rather than an oracle
app. We would agree to that in a vacuum, but the reality of the situation is that the ecosystem
needs these events and getters to build upon, and we're not upgrading the core contract anytime
soon. So we're taking a workaround and putting some code that best belongs elsewhere in the oracle.

## Add getters for accessing the current state details.

In addition to the getters listed in sections above, the following functions provide public access
to the current reporting state.

    function getCurrentOraclesReportStatus() public view returns(uint256)

    function getCurrentReportKindsSize() public view returns(uint256)

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
