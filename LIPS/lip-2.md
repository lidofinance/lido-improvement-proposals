---
lip: 2
title: Oracle contract upgrade to v2
status: Work in progress
author: Denis Glotov
discussions-to: none
created: 2021-02-22
updated: 2021-12-23
---

# Oracle contract upgrade to v2

The following new features are added to the first version of the oracle contract.


## Allow the number of oracle members to be less than the quorum value.

The most likely reason for removing an oracle member is a malicious oracle. It's better to have an
oracle with no quorum (as members are added or quorum value lowered by the voting process) than an
oracle with a quorum but a malicious member in it.

We implemented this by updating the restrictions (were `members.length >= _quorum`). Now the only
way to update the quorum value is with its mutator:

    function setQuorum(uint256 _quorum) external auth(MANAGE_QUORUM)


## Sanity checks the oracles reports by configurable values.

In order to limit the misbehaving oracles impact, we want to limit oracles report change by 0.1 APR
increase in stake and 15% decrease in stake. Both values are configurable by DAO voting in case of
extremely unusual circumstances.

Note that the change is evaluated after the quorum of oracles reports is reached, and not on the
individual report.

For this, we have added the following accessors and mutators:

    function getAllowedBeaconBalanceAnnualRelativeIncrease() public view returns(uint256)

    function getAllowedBeaconBalanceRelativeDecrease() public view returns(uint256)

    function setAllowedBeaconBalanceAnnualRelativeIncrease(uint256 value)
        public auth(SET_REPORT_BOUNDARIES)

    function setAllowedBeaconBalanceRelativeDecrease(uint256 value)
        public auth(SET_REPORT_BOUNDARIES)

And the logic of reporting to the Lido contract got a call to `_reportSanityChecks` that does the
following. It compares the total pooled ether, grabbed from the Lido contract, right before and
after reporting the quorum report, and

* if there is a profit or same, calculates the [APR][1], compares it with the upper bound. If was above,
  reverts the transaction with `ALLOWED_BEACON_BALANCE_INCREASE` code.
* if there is a loss, calculates relative decrease and compares it with the lower bound. If was
  below, reverts the transaction with `ALLOWED_BEACON_BALANCE_DECREASE` code.

[1]: https://en.wikipedia.org/wiki/Annual_percentage_rate


## Use only one epoch per frame to simplify vote accounting.


## Add calculation of rewards APR.


## Callback function to be invoked every time the quorum is reached among oracle daemons data.


## Add events to cover all states change and getters for accessing the current state details.
